# PopIn-PaymentForm-Angular

## Índice

- [1. Introducción](#1-introducción)
- [2. Requisitos previos](#2-requisitos-previos)
- [3. Despliegue](#3-despliegue)
- [4. Datos de conexión](#4-datos-de-conexión)
- [5. Transacción de prueba](#5-transacción-de-prueba)
- [6. Implementación de la IPN](#6-implementación-de-la-ipn)
- [7. Personalización](#7-personalización)
- [8. Consideraciones](#8-consideraciones)

## 1. Introducción

En este manual podrás encontrar una guía paso a paso para configurar un proyecto de **[Angular]** con la pasarela de pagos de IZIPAY. Te proporcionaremos instrucciones detalladas y credenciales de prueba para la instalación y configuración del proyecto, permitiéndote trabajar y experimentar de manera segura en tu propio entorno local.
Este manual está diseñado para ayudarte a comprender el flujo de la integración de la pasarela para ayudarte a aprovechar al máximo tu proyecto y facilitar tu experiencia de desarrollo.

> [!IMPORTANT]
> En la última actualización se agregaron los campos: **nombre del tarjetahabiente** y **correo electrónico** (Este último campo se visualizará solo si el dato no se envía en la creación del formtoken).

<p align="center">
  <img src="https://github.com/izipay-pe/Imagenes/blob/main/formulario_popin/Imagen-Formulario-Popin.png?raw=true" alt="Popin" width="250"/>
</p>

<a name="Requisitos_Previos"></a>

## 2. Requisitos previos

- Comprender el flujo de comunicación de la pasarela. [Información Aquí](https://secure.micuentaweb.pe/doc/es-PE/rest/V4.0/javascript/guide/start.html)
- Extraer credenciales del Back Office Vendedor. [Guía Aquí](https://github.com/izipay-pe/obtener-credenciales-de-conexion)
- Debe instalar la [versión de LTS node.js](https://nodejs.org/es/).
- Para este proyecto utilizamos la herramienta Visual Studio Code.
> [!NOTE]
> Tener en cuenta que, para que el desarrollo de tu proyecto, eres libre de emplear tus herramientas preferidas.

## 3. Despliegue

### Clonar el proyecto:

```sh
git clone [https://github.com/izipay-pe/PopIn-PaymentForm-Angular.git]
```

### Ejecutar proyecto

- Ingrese a la carpeta raiz del proyecto.

- A continuación, instale el cliente angular-cli:

  ```bash
  npm install -g @angular/cli
  ```

  Más detalles en la página web de [angular-cli](https://angular.io/guide/quickstart).

- Agregue la dependencia con:

  ```bash
  npm install --save @lyracom/embedded-form-glue
  ```

Ejecútelo y pruébelo en minimal-example:

```sh
npm run start
```

ver el resultado en http://localhost:4200/

## 4. Datos de conexión

**Nota**: Reemplace **[CHANGE_ME]** con sus credenciales de `API REST` extraídas desde el Back Office Vendedor, ver [Requisitos Previos](#Requisitos_Previos).

- Editar en src/index.html en la sección HEAD.

  ```javascript
  <!-- tema y plugins. debe cargarse en la sección HEAD -->
  <link rel="stylesheet"
  href="~~CHANGE_ME_ENDPOINT~~/static/js/krypton-client/V4.0/ext/classic-reset.css">
  <script
      src="~~CHANGE_ME_ENDPOINT~~/static/js/krypton-client/V4.0/ext/classic.js">
  </script>
  ```

- Edita src/app/app.component.html template, se agregará al elemento #myPaymentForm para ver formulario de pago.

  ```html
  <div class="form">
    <h1>{{ title }}</h1>
    <div class="container">
      <div id="myPaymentForm"></div>
    </div>
  </div>
  ```

- Actualice los estilos en src/app/app.component.css.

  ```css
  h1 {
    margin: 40px 0 20px 0;
    width: 100%;
    text-align: center;
  }
  .container {
    display: flex;
    justify-content: center;
  }
  ```

- Edite el componente predeterminado src/app/app.component.ts, con el siguiente codigo si quiere interactuar con el formulario de pago, con un endpoint propio.

  ```js
  import { HttpClient,HttpHeaderResponse,HttpHeaders } from "@angular/common/http";
  import { Component, AfterViewInit, ChangeDetectorRef} from "@angular/core";
  import KRGlue from "@lyracom/embedded-form-glue";
  import { firstValueFrom } from "rxjs";

  @Component({
    selector: "app-root",
    templateUrl: "./app.component.html",
    styleUrls: ["./app.component.css"]
  })
  export class AppComponent implements AfterViewInit{
    title: string = "Ejemplo de un formulario popin en ANGULAR";
    message: string = "";

    constructor(
      private http: HttpClient,
      private chRef: ChangeDetectorRef
    ){}

    ngAfterViewInit(){

      /* llenar con las credenciales extraídas del Back Office Vendedor para mas detalle regresar a:  Requisitos Previos. */

      const endpoint = "~~CHANGE_ME_ENDPOINT~~";
      const publicKey = "~~CHANGE_ME_PUBLIC_KEY~~";
      const formToken = ""; /* solo está declarando la variable dejar vacío */

      /* variable que recibe el monto a pagar */
      let monto = 80;

      /* arreglo que enviara la data al endpoint */
      const data = {
                    amount: monto*100,
                    currency: 'PEN'
      };

      /* abre una nueva conexión, utilizando la solicitud POST en el URL de su endpoint */
      const observable = this.http.post("YOUR_SERVER/payment/init", data,{responseType: 'text'});

      firstValueFrom(observable).then((resp: any) => {
        formToken = resp
        return KRGlue.loadLibrary(endpoint, publicKey) /* cargar la libreria KRGlue */
      })
      .then(({ KR }) =>
        KR.setFormConfig({
          /* establecer la configuración mínima */
          formToken: formToken,
          "kr-language": "es-ES" /* cambia el idioma del formulario */
        })
      )
      .then(({ KR }) => KR.onSubmit(this.onSubmit))
      .then(({ KR }) => KR.addForm("#myPaymentForm")) /* agregar un formulario de pago a myPaymentForm div */
      .then(({ KR, result }) => KR.showForm(result.formId)) /* muestra el formulario de pago */
      .catch(
        error => {
          this.message = error.message + " (see console for more details)";
        }
      );
    }
  }
  ```

- ## 4.1.- Validando el pago (Opcional)

  El hash de pago debe validarse en el lado del servidor para evitar la exposición de su clave hash personal.

  - En el lado del servidor:

    ```js
      const express = require('express')
      const hmacSHA256 = require('crypto-js/hmac-sha256')
      const Hex = require('crypto-js/enc-hex')
      const app = express()
      (...)
      // válida los datos de pago dados (hash)
      app.post('/validatePayment', (req, res) => {
        const answer = req.body.clientAnswer
        const hash = req.body.hash
        const answerHash = Hex.stringify(
          hmacSHA256(JSON.stringify(answer), 'CHANGE_ME: HMAC SHA256 KEY')
        )
        if (hash === answerHash) res.status(200).send('Valid payment')
        else res.status(500).send('Payment hash mismatch')
      })
      (...)
    ```

  - Del lado del cliente:

    ```js
      import { Component, OnInit } from "@angular/core";
      import KRGlue from "@lyracom/embedded-form-glue";
      import axios from 'axios'
      @Component({
        selector: "app-root",
        templateUrl: "./app.component.html",
        styleUrls: ["./app.component.css"]
      })
      export class AppComponent implements OnInit {
        title: string = "Ejemplo de un formulario popin en ANGULAR";
        (...),
          ngOnInit() {
            /* use su endpoint y la clave public_key */
            const endpoint = '~~CHANGE_ME_ENDPOINT~~'
            const publicKey = '~~CHANGE_ME_PUBLIC_KEY~~'
            const formToken = 'DEMO-TOKEN-TO-BE-REPLACED'
            KRGlue.loadLibrary(endpoint, publicKey) /* cargar la libreria KRGlue */
              .then(({KR}) => KR.setFormConfig({  /* establecer la configuración mínima */
                formToken: formToken,
                'kr-language': 'en-US',
              })) /* para actualizar el parámetro de inicialización */
              .then(({KR}) => KR.onSubmit(resp => {
                axios
                  .post('http://localhost:3000/validatePayment', paymentData)
                  .then(response => {
                    if (response.status === 200) this.message = 'Payment successful!'
                  })
                return false
              }))
              .then(({KR}) => KR.addForm('#myPaymentForm')) /* crear un formulario de pago */
              .then(({KR, result}) => KR.showForm(result.formId));  /* muestra el formulario de pago */
          }
          (...)
      }
    ```

## 5. Transacción de prueba

Antes de poner en marcha su pasarela de pago en un entorno de producción, es esencial realizar pruebas para garantizar su correcto funcionamiento.

Puede intentar realizar una transacción utilizando una tarjeta de prueba con la barra de herramientas de depuración (en la parte inferior de la página).

<p align="center">
  <img src="https://i.postimg.cc/3xXChGp2/tarjetas-prueba.png" alt="Formulario"/>
</p>

- También puede encontrar tarjetas de prueba en el siguiente enlace. [Tarjetas de prueba](https://secure.micuentaweb.pe/doc/es-PE/rest/V4.0/api/kb/test_cards.html)

## 6. Implementación de la IPN

> [!IMPORTANT]
> Es recomendable implementar la IPN para comunicar el resultado de la solicitud de pago al servidor del comercio.

La IPN es una notificación de servidor a servidor (servidor de Izipay hacia el servidor del comercio) que facilita información en tiempo real y de manera automática cuando se produce un evento, por ejemplo, al registrar una transacción.
Los datos transmitidos en la IPN se reciben y analizan mediante un script que el vendedor habrá desarrollado en su servidor.

- Ver manual de implementación de la IPN. [Aquí](https://secure.micuentaweb.pe/doc/es-PE/rest/V4.0/kb/payment_done.html)
- Vea el ejemplo de la respuesta IPN con PHP. [Aquí](https://github.com/izipay-pe/Server-IPN-Php)
- Vea el ejemplo de la respuesta IPN con NODE.JS. [Aquí](https://github.com/izipay-pe/Server-IPN-JavaScript)

## 7. Personalización

Si deseas aplicar cambios específicos en la apariencia de la pasarela de pago, puedes lograrlo mediante la modificación de código CSS. En este enlace [Código CSS - Pop-in](https://github.com/izipay-pe/Personalizacion/blob/main/Formulario%20Popin/Style-Personalization-PopIn.css) podrá encontrar nuestro script para un formulario pop-in.

## 8. Consideraciones

Para obtener más información, echa un vistazo a:

- [Formulario incrustado: prueba rápida](https://secure.micuentaweb.pe/doc/es-PE/rest/V4.0/javascript/quick_start_js.html)
- [Primeros pasos: pago simple](https://secure.micuentaweb.pe/doc/es-PE/rest/V4.0/javascript/guide/start.html)
- [Servicios web - referencia de la API REST](https://secure.micuentaweb.pe/doc/es-PE/rest/V4.0/api/reference.html)
