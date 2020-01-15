
# Python-example

Ejemplo de Timbrado con Delphi
<br/>

## Requerimientos
* [Python version 2.7.9 o superior](https://www.python.org/downloads/)

* Librerías
   * [Zeep: Python SOAP client](https://python-zeep.readthedocs.io/en/master/)
     >  ``` pip install zeep ```

   * [pyOpenSSL](https://www.pyopenssl.org/en/stable/install.html)
     > ``` pip install pyopenssl ```
   * [PyCryptodome](https://pycryptodome.readthedocs.io/en/latest/src/installation.html)

     >  ``` pip install pycryptodome ```

<br/>

# Ejemplo de timbrado

```Python
usuario='pruebasWS'
password='pruebasWS'
path = os.path.dirname(os.path.abspath(__file__)) + '/resources/'

file_xml = path + 'xml.xml'
cert_path = path + 'CSD_Pruebas_CFDI_LAN7008173R5.cer'
key_path = path + 'CSD_Pruebas_CFDI_LAN7008173R5.key.pem'
cf = ClienteFormas(file_xml, cert_path, key_path)
# Sellado de documento
cfdi_sellado = cf.sellar()
# Timbrado de documento
documento_timbrado = cf.timbrar(usuario,password,cfdi_sellado)
# Guardar respuesta WS documento como XML
cf.save_xml(path, 'xml_sellado.xml', documento_timbrado['xmlTimbrado'])
```

<br>

La clase **ClienteFormas** contiene el código para poder hacer el sellado del CFDi, y también el código que nos permite cargar los archivos necesarios para hacer el sellado.

<br/>

## Funciones de la clase ClienteFormas


### def sellar(self):
Contiene la funcion llamada get_sello la cual recibe el archivo XML que queremos sellar, el certificado y la llave privada para poder crear el sello y enviarlo al servicio web (dentro de esta funcion hacemos las llamadas a las demas funciones para generar la cadena original, el sello, y enviarlo al servicio web).


### def get_cadena_original(self, xml=None):
A esta función le enviamos el documento XML para que nos genere la cadena original del documento la cual utilizaremos para generar el sello del CFDi.

### def get_no_certificado(self, serial_hex):
Esta función nos regresa la información del certificado el serialNumber que asignaremos a nuestro XML al atributo NoCertificado para que sea posible hacer el timbrado.


### def get_certificado_64(self):
Nos regresa el contenido del certificado en base64.

### def timbrar(self, usuario, password, cfdi_sellado):
Se le envían los datos de acceso al WebService (usuario y contraseña), y el XML sellado. Si todo es correcto el webservice nos retornará un nodo llamado xmlTimbrado el cual contiene nuestro CFDi timbrado.

### def save_xml(self, path, filename, xml):
Se le envían datos de ruta, nombre de archivo y el xml timbrado para almacenarlo en un nuevo archivo XML.



```Python
# coding=utf-8

import zeep
import os
import re
from lxml import etree as ET
from datetime import datetime
from OpenSSL import crypto
from cryptography import x509
from cryptography.hazmat.backends import default_backend
from cryptography.hazmat.primitives.serialization import load_ssh_public_key, load_pem_private_key
from cryptography.hazmat.primitives.serialization import Encoding
from cryptography.hazmat.primitives.asymmetric import padding
from cryptography.hazmat.primitives import hashes
from cryptography.hazmat.primitives.serialization import PublicFormat
import base64
from Crypto.Hash import SHA256
from Crypto.Signature import PKCS1_v1_5
from Crypto.PublicKey import RSA
import codecs


class ClienteFormas:

    # CARGAR XSLT CADENA ORIGINAL (SITIO SAT)
    xlst_path = "http://www.sat.gob.mx/sitio_internet/cfd/3/cadenaoriginal_3_3/cadenaoriginal_3_3.xslt"
    
    # CARGAR XSLT CADENA ORIGINAL (LOCALMENTE)
    # xlst_path = os.path.dirname(os.path.abspath(__file__)) + '/resources/cadenaoriginal_3_3.xslt'
    
    # WSDL PRUEBAS 	 (FORMAS DIGITALES)
    wsdl_url = "http://dev33.facturacfdi.mx/WSTimbradoCFDIService?wsdl" 
    
    # WSDL PRODUCCIÓN (FORMAS DIGITALES)
    #wsdl_url="https://v33.facturacfdi.mx/WSTimbradoCFDIService?wsdl"

    BEGIN_CERTIFICATE = '-----BEGIN CERTIFICATE-----\n'
    END_CERTIFICATE = '\n-----END CERTIFICATE-----\n'
    DEBUG = False

 
    def __init__(self, xml, certificado, llave_privada, Debug=None):
        self.certificado=certificado
        self.xml=xml
        self.llave_privada = llave_privada
        if Debug is not None:
            self.DEBUG = Debug
    

    def timbrar(self, usuario, password, cfdi_sellado):
        cliente = zeep.Client(wsdl = self.wsdl_url)
        try:
            #timbrar cfdi
            accesos_type = cliente.get_type("ns1:accesos")
            
            accesos = accesos_type(usuario=usuario, password=password)
            cfdi_timbrado = cliente.service.TimbrarCFDI(accesos = accesos, comprobante=cfdi_sellado.decode('utf-8'))
            if self.DEBUG:
                print(cfdi_sellado.decode('utf-8'))
                print(cfdi_timbrado) 
            return cfdi_timbrado  
        except Exception as exception:
            print("Message %s" % exception)


    def sellar(self):
        #print(self.xml)
        # Insertar fecha al documento (Opcional)
        fecha = str(datetime.now().isoformat())[:19]
        
        tmpxml = ET.parse(self.xml).getroot()

        certificado64 = self.get_certificado_64()
        certificado = self.get_certificado_x509(certificado64)
        no_certificado = self.get_no_certificado(hex(certificado.get_serial_number()))

        # Insertar fecha al documento (Opcional)
        tmpxml.attrib['Fecha'] = fecha
        tmpxml.attrib['NoCertificado'] = no_certificado
        tmpxml.attrib['Certificado'] = certificado64
        
        cadena_original = self.get_cadena_original(tmpxml)

        sello = self.get_sello(cadena_original)

        if self.DEBUG:
            print('cadena original:  {0} \n Sello {1} \n Certificado {2} No de Certificado {3}\n'.format(cadena_original, sello, certificado64, no_certificado))

        tmpxml.attrib['Sello'] = sello

        return ET.tostring(tmpxml)


    def get_certificado_64(self):
        with open(self.certificado, 'rb') as f:
            cerfile = f.read()

        cert64 = base64.b64encode(cerfile)
        return cert64.decode('utf-8')


    def get_certificado_x509(self, certificado_base_64):
        cert64Str = re.sub("(.{64})", "\\1\n", certificado_base_64, 0, re.DOTALL)
        cert64WithBE = self.BEGIN_CERTIFICATE + cert64Str + self.END_CERTIFICATE
        certificate = crypto.load_certificate(crypto.FILETYPE_PEM, cert64WithBE.encode("utf-8"))
        return certificate

    def get_no_certificado(self, serial_hex):
        no_certificado = ''
        serial_str = str(serial_hex)
        if (len(serial_str) % 2) == 1:
            serial_str = ' ' + serial_str

        serial_len = len(serial_str) // 2

        for index in range(serial_len):
            start_index = (index * 2) 
            end_index = start_index + 2
            aux = serial_str[start_index:end_index]
            
            if 'x' != aux[1]:
                no_certificado = no_certificado + aux[1]
           

        return no_certificado

    def get_cadena_original(self, xml=None):
        #current_cfdi = ET.parse(file_xml)
        xslt = ET.parse(self.xlst_path)
        transform = ET.XSLT(xslt)
        cadena_original = transform(xml)
        return (str(cadena_original))


    def get_sello(self, cadena_original):
        private_key = False
        with open(self.llave_privada, "r") as myfile:
            private_key = RSA.importKey(myfile.read())

        digest2 = SHA256.new()
        digest2.update(str(cadena_original).encode('utf-8'))

        signer = PKCS1_v1_5.new(private_key)
        sig = signer.sign(digest2)
        return base64.b64encode(sig).decode('utf-8')

    def save_xml(self, path, filename, xml):
    	file = codecs.open(path + filename, "w", "utf-8")
    	file.write(xml)
    	file.close()
    



#Ejemplo de timbrado
usuario='pruebasWS'
password='pruebasWS'
path = os.path.dirname(os.path.abspath(__file__)) + '/resources/'

file_xml = path + 'xml.xml'
cert_path = path + 'CSD_Pruebas_CFDI_LAN7008173R5.cer'
key_path = path + 'CSD_Pruebas_CFDI_LAN7008173R5.key.pem'
cf = ClienteFormas(file_xml, cert_path, key_path)

# Sellado de documento
cfdi_sellado = cf.sellar()

# Timbrado de documento
documento_timbrado = cf.timbrar(usuario,password,cfdi_sellado)

# Guardar respuesta WS documento como XML
cf.save_xml(path, 'xml_sellado.xml', documento_timbrado['xmlTimbrado'])

```
