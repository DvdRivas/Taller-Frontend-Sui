![banner](./imagenes/banner.jpg)
# Taller: Interacción con la blockchain de Sui desde el frontend

Sui es una plataforma de blockchain y contratos inteligentes de capa 1 diseñada para que la propiedad de activos digitales sea rápida, privada, segura y accesible.

Move es un lenguaje de código abierto para escribir paquetes seguros para manipular objetos en blockchain. Permite bibliotecas, herramientas y comunidades de desarrolladores comunes en blockchains con modelos de datos y ejecución muy diferentes.

## Proyecto base

Puedes usar este repositorio como tu punto de despliegue de tu proyecto backend. El proyecto base es el de **Mi-Empresa-De-**, que te permite crear bases de datos para empresas que ofrece un sistema de gestion de clientes basado en niveles.

### Abriendo con Codespaces

* Asegúrate de clonar este repositorio a tu cuenta usando el botón **`Fork`**.
    
    ![fork](./imagenes/fork.png)
    * Puedes renombrar el repositorio a lo que sea que se ajuste con tu proyecto.
* Presiona el botón **`<> Code`** y luego haz click en la sección **`Codespaces`**

    ![codespaces](./imagenes/codespaces.png)

* Por último, presiona **`Create codespace on master`**. Esto abrirá el proyecto en una interfaz gráfica de Visual Studio Code e instalará todas las herramientas necesarias para desarrollar con Move.

## Configuración inicial para despliegue en testnet

Este repositorio cuenta con un archivo de automatización que permite al usuario obtener su address y entorno de testnet de forma autómatica. Para consultar el address es necesario ejecutar el siguiente comando:
```
sui client addresses
```

## Fondeando una cuenta

Una vez conectado, el siguiente paso es **fondear tu cuenta**, es decir, asegurarte de que la dirección que estás utilizando tenga **tokens SUI** (aunque sean tokens de prueba) suficientes para cubrir las tarifas de las transacciones. Este proceso es esencial para poder desplegar paquetes, ejecutar funciones y validar tu lógica en cualquier red que estés utilizando.

Puedes hacer esto ejecutando el siguiente comando en tu terminal:

```sh
sui client faucet
```

Obtendrás algo similar a esto, probablemente en letras rojas:

```sh
For testnet tokens, please use the Web UI: https://faucet.sui.io/?address=0x451ef911c5a1706d4833f89b75f6cb49c55a586821e9b7de6bd9d8b41dac2cda
```
Puedes hacer click en esa URL, la cual te llevará al faucet de Sui, que es una aplicación que reparte tokens de prueba en las redes `testnet` y `devnet`, para que los desarrolladores puedan desplegar y probar sus paquetes Move.

![faucet](imagenes/testnetfaucet.png)

Ya en el sitio, simplemente haz click en **Request Testnet SUI**. Con esto habremos terminado el proceso de fondeo. Puedes verificarlo en terminal

```sh
sui client balance
╭────────────────────────────────────────╮
│ Balance of coins owned by this address │
├────────────────────────────────────────┤
│ ╭──────────────────────────────────╮   │
│ │ coin  balance (raw)  balance     │   │
│ ├──────────────────────────────────┤   │
│ │ Sui   10000000000    10.00 SUI   │   │
│ ╰──────────────────────────────────╯ │
╰────────────────────────────────────────╯
```

> :information_source: El README distorsiona un poco este output.

Puedes acceder al faucet directamente desde acá: https://faucet.sui.io/

## Desplegando el proyecto en la testnet
La testnet es un entorno de pruebas creado para que los desarrolladores de la blockchain Sui experimenten e interactuen con sus paquetes antes de subirlos de manera oficial a la Mainnet, la red real. En esta ocasión realizaremos el despliegue del paquete con el que interactuaremos en el frontend en la testnet.

Una vez recibidos los tokens de testnet mediante el faucet, desplegar a la testnet es muy sencillo, solo es necesario ejecutar el comando:
```
sui client publish
```

Lo que dará como resultado mucha informacion relacionada con la transaccion. Sin embargo, la información en la que nos vamos a centrar es la siguiente: 

```
╭──────────────────────────────────────────────────────────────────────────────────────────────────╮
│ Object Changes                                                                                   │
├──────────────────────────────────────────────────────────────────────────────────────────────────┤
│ Created Objects:                                                                                 │
│  ┌──                                                                                             │
│  │ ObjectID: 0xeee71d904b4ba170d130dacdaf5de7eabd23a6d3634fd39e06ceb6e2e630522b                  │
│  │ Sender: 0xfdfb28de3b66e3d21922ed3a1f13cb99b5c7d848264fab94358d17e76647b6a0                    │
│  │ Owner: Account Address ( 0xfdfb28de3b66e3d21922ed3a1f13cb99b5c7d848264fab94358d17e76647b6a0 ) │
│  │ ObjectType: 0x2::package::UpgradeCap                                                          │
│  │ Version: 349180416                                                                            │
│  │ Digest: HuYaAgoVDsdbt3skuHzBgstvupEqL6AjEURbPgjb86sn                                          │
│  └──                                                                                             │
│ Mutated Objects:                                                                                 │
│  ┌──                                                                                             │
│  │ ObjectID: 0x12b4b6e610179a45e4e5824a5189b73bbd88d37adc8e91988ce343689172ba45                  │
│  │ Sender: 0xfdfb28de3b66e3d21922ed3a1f13cb99b5c7d848264fab94358d17e76647b6a0                    │
│  │ Owner: Account Address ( 0xfdfb28de3b66e3d21922ed3a1f13cb99b5c7d848264fab94358d17e76647b6a0 ) │
│  │ ObjectType: 0x2::coin::Coin<0x2::sui::SUI>                                                    │
│  │ Version: 349180416                                                                            │
│  │ Digest: 9pvSJgWuu2Ki7qz5ZKRiarP7XwEHSkXYBnj5RGrPkkF6                                          │
│  └──                                                                                             │
│ Published Objects:                                                                               │
│  ┌──                                                                                             │
│  │ PackageID: 0x8ddace66e376f03067016c51820d512fa1a8fa9e2e518ed0c842086cdb27ae91                 │
│  │ Version: 1                                                                                    │
│  │ Digest: 71yxJBgson9NHunqDvmKiuM5XVfsaszxwhdnXCPTsbeX                                          │
│  │ Modules: empresa                                                                              │
│  └──                                                                                             │
╰──────────────────────────────────────────────────────────────────────────────────────────────────╯
```

Donde al final se muestra el PackageID, o ID del paquete (nuestro código). Es importante guardar este ID, ya que mediante su uso es como haremos interaccion con nuestro paquete.

## 


> Este repositorio fue creado con base al sui-starter-kit de ZinHunter: https://github.com/WayLearnLatam/sui-starter-kit
