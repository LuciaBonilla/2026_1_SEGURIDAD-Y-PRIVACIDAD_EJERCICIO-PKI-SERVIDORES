# 2026_1_SEGURIDAD-Y-PRIVACIDAD_EJERCICIO-PKI-SERVIDORES

## Contexto

En mTLS intervienen dos certificados distintos firmados por la misma CA raíz (CN=SeguridadYPrivacidad):

- El certificado de servidor (CN=LUCIA-BONILLA) prueba la identidad del servidor ante el navegador.

- El certificado de cliente (CN=lucia.bonilla@correo.ucu.edu.uy) prueba la identidad del usuario ante nginx. Lo presenta el navegador a nginx.

Y nginx necesita un tercer insumo: el certificado público de la CA raíz, para poder verificar que el certificado que presenta el cliente realmente fue emitido por CN=SeguridadYPrivacidad.

|Archivo                                            |Ubicación en el host               |Dónde se usa               |
| --------                                          | -------                           | -------                   |
|server.crt + server.key                            |./certs/server/                    |nginx (servidor)       |
|SeguridadYPrivacidadCA.cer                         |./certs/ca/                        |nginx (valida clientes)    |
|client.key + client.csr + client.crt + client.p12  |./client/ (No van al contenedor)   |Se instalan en el navegador|


## Pasos seguidos

Los pasos seguidos son:

### Paso 1 — Certificado de la CA

Obtenido desde la Webasignatura; está en ./certs/ca/

### Paso 2 — Certificado de servidor

Generar la clave privada RSA 2048 (sin contraseña, para que nginx arranque sin pedir contraseña) y el CSR.

Solicitud de certificado de servidor (CN = LUCIA-BONILLA):

```bash
openssl req -new -newkey rsa:2048 -noenc \
  -keyout certs/server/server.key \
  -out certs_requests/server.csr
```

C=UY
O=UCU
CN=LUCIA-BONILLA

-noenc genera la clave sin contraseña. Esto es deliberado: si la clave del servidor tuviera contraseña, nginx pediría la contraseña en cada arranque y el contenedor no levantaría desatendido.

El certificado digital de servidor fue obtenido del foro de la Webasignatura: https://webasignatura.ucu.edu.uy/mod/forum/discuss.php?d=274398 

### Paso 3 — Certificado de cliente

Generar la clave privada RSA 2048 (sin passphrase, para que nginx arranque sin pedir contraseña) y el CSR.

Solicitud de certificado de cliente (CN = lucia.bonilla@correo.ucu.edu.uy):

```bash
openssl req -new -newkey rsa:2048 \
  -keyout client/client.key \
  -out certs_requests/client.csr
```

C=UY
O=UCU
CN=lucia.bonilla@correo.ucu.edu.uy
pide contraseña

El certificado digital de cliente fue obtenido del foro de la Webasignatura: https://webasignatura.ucu.edu.uy/mod/forum/discuss.php?d=274398 

### Paso 4 — Empaquetar la identidad del cliente en PKCS#12 para importarlo al navegador

Los navegadores importan certificados de cliente como .p12/.pfx, que agrupan certificado + clave privada:

```bash
openssl pkcs12 -export \
  -inkey client/client.key \
  -in client/lucia.bonilla@correo.ucu.edu.uy.crt \
  -certfile certs/ca/SeguridadYPrivacidadCA.cer \
  -out client/lucia.bonilla@correo.ucu.edu.uy.p12
```

Pide una contraseña; el navegador la pedirá al importar.

### Paso 5 — Importar certificado de cliente en el navegador

En este caso, se importó en Windows al abrir el archivo .p12 e ingresar la contraseña.

### Paso 6 — Importar certificado de CA raíz en el navegador

En este caso, se importó el certificado en el navegador Brave:


Paso 7 — Levantar el contenedor

```bash
docker compose up -d          # levantar
docker compose logs -f        # ver errores de arranque (útil si nginx no valida los certs)
docker compose down           # detener
docker compose restart        # reiniciar tras cambiar nginx.conf o certificados
```

Paso 8 — Verificar