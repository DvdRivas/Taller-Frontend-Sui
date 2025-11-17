# Imports
```js
import { useSuiClient, useSignAndExecuteTransaction } from "@mysten/dapp-kit";
import { Transaction } from "@mysten/sui/transactions";
import { isValidSuiObjectId } from "@mysten/sui/utils";
import { useNetworkVariable } from "./networkConfig";
import { ConnectButton, useCurrentAccount } from "@mysten/dapp-kit";
```
* useSuiClient()
Crea un cliente conectado al RPC de Sui; se usa para inspecciones y consultas.

* useSignAndExecuteTransaction()
Permite firmar y enviar transacciones reales con la wallet conectada.

* Transaction
Constructor de bloques de transacción (Move calls, transferencias, etc.)

* isValidSuiObjectId
Utilidad para validar ObjectIDs (0x...).

* useNetworkVariable
 Utilidad personalizada para obtener valores de configuración según red (ej. packageId).

* useCurrentAccount / ConnectButton
Manejo de conexión de wallet y cuenta activa.
# Variables 
```js
  const suiClient = useSuiClient()
  const cuenta = useCurrentAccount()
  const { mutate: signAndExecute } = useSignAndExecuteTransaction()
  const [estado, cambiarEstado] = useState(false);
  const [respuesta, cambiarRespuesta] = useState(null);
  const [nuevaEmpresa, setNuevaEmpresa] = useState(false)
  const [objectId, setObjectId] = useState(() => {
    const hash = window.location.hash.slice(1);
    return isValidSuiObjectId(hash) ? hash : null;
  });
  const packageId = useNetworkVariable("PackageId");
  const modulo = "empresa"
```
* suiClient Genera una instancia con el cliente de Sui.
* cuenta Carga los detalles de la cuenta de slush, por ejemplo, el address.
* signAndExecute Funcion para poder realizar la operacion de confirmar una transaccion. Vital para transacciones que modifican o escriben datos.
* useStates Variables que al modificarse forzan una renderizacion de la pagina y permite mostrar los cambios realizados.

# ClientCall
```js
  async function ClientCall(params) {
    
    cambiarEstado(true);

    try {
      cambiarRespuesta(null);

      const tx = new Transaction();

      // ===============================
      // 1) SERIALIZACIÓN DE ARGUMENTOS
      // ===============================
      const args = params.args.map((arg, idx) => {
        console.log("ARG RAW", idx, arg);

        // 1. ObjectID válido → tx.object(...)
        if (typeof arg === "string" && isValidSuiObjectId(arg)) {
          console.log(`Arg[${idx}] es ObjectID → tx.object(${arg})`);
          return tx.object(arg);
        }

        // 2. Tipos explícitos { type, value }
        if (arg && typeof arg === "object" && "type" in arg) {
          const { type, value } = arg;

          switch (type) {
            case "u8": return tx.pure.u8(Number(value));
            case "u16": return tx.pure.u16(Number(value));
            case "u32": return tx.pure.u32(Number(value));
            case "u64": return tx.pure.u64(BigInt(value));
            case "u128": return tx.pure.u128(BigInt(value));
            case "bool": return tx.pure.bool(Boolean(value));
            case "string": return tx.pure.string(String(value));
            case "address": return tx.pure.address(value);
            default:
              console.warn(`Tipo no manejado (${type}), usando tx.pure`);
              return tx.pure(value);
          }
        }

        // 3. Inferencias básicas
        if (typeof arg === "boolean") return tx.pure.bool(arg);
        if (typeof arg === "number") return tx.pure.u64(BigInt(arg));
        if (typeof arg === "bigint") return tx.pure.u64(arg);
        if (typeof arg === "string") return tx.pure.string(arg);

        // 4. Fallback
        console.warn(`Arg[${idx}] fallback → tx.pure(arg)`);
        return tx.pure(arg);
      });

      console.log("ARGS FINAL →", args);

      // ===============================
      // 2) CONSTRUIR MOVE CALL
      // ===============================
      tx.moveCall({
        target: `${packageId}::${modulo}::${params.funcion}`,
        arguments: args,
      });

      console.log("TARGET:", `${packageId}::${modulo}::${params.funcion}`);

      // ====================================
      // 3) DETECTAR SI ES FUNCIÓN "VIEW"
      // ====================================
      const esLectura =
        params?.soloLectura === 1 ||
        params?.soloLectura === "1" ||
        params?.soloLectura === true ||
        params?.soloLectura === "true";

      // =======================================================
      // CASE 1: FUNCIÓN DE SOLO LECTURA → devInspect
      // =======================================================
      if (esLectura) {
        console.log("FUNCIÓN VIEW → ejecutando devInspect…");

        const result = await suiClient.devInspectTransactionBlock({
          sender: cuenta.address,
          transactionBlock: tx,
        });

        console.log("devInspect result:", result);

        const decoded = decodeReturnValues(result);
        cambiarRespuesta(decoded);

        return decoded;
      }

      // =======================================================
      // CASE 2: TRANSACCIÓN REAL
      // =======================================================
      console.log("FUNCIÓN MUTANTE → firmando transacción…");

      signAndExecute(
        { transaction: tx },
        {
          onSuccess: async (txres) => {
            const result = await suiClient.waitForTransaction({
              digest: txres.digest,
              options: { showEffects: true, showEvents: true },
            });

            console.log("RESULTADO EJECUCIÓN:", result);

            const decoded = decodeReturnValues(result);
            if (decoded !== null) cambiarRespuesta(decoded);

            // Si se creó una empresa, actualizar ID
            if (params.funcion === "crear_empresa") {
              const id = result.effects?.created?.[0]?.reference?.objectId;
              if (id) {
                setObjectId(id);
                window.location.hash = id;
                setNuevaEmpresa(true);
              }
            }
          },
          onError: (err) => {
            alert("Error al enviar transacción: " + err.message);
          },
        }
      );
    } catch (error) {
      alert("Hubo un error: " + error.message);
      console.error(error);
    } finally {
      cambiarEstado(false);
    }
  }

  //
  // ========================================
  // DECODIFICADOR UNIVERSAL DE RETURN VALUES
  // ========================================
  //
  function decodeReturnValues(result) {
    try {
      const values =
        result.results?.[0]?.returnValues ||
        result.effects?.returnValues;

      if (!values || values.length === 0) return null;

      const decoded = values.map(([bytes, typeTag]) => {
        return decodeByType(bytes, typeTag);
      });

      return decoded.length === 1 ? decoded[0] : decoded;
    } catch (err) {
      console.warn("decodeReturnValues ERROR:", err);
      return null;
    }
  }

  //
  // ===========================
  // DECODIFICACIÓN POR TIPO
  // ===========================
  //
  function decodeByType(bytes, typeTag) {
    const arr = Uint8Array.from(bytes);

    if (!typeTag) return null;

    // PRIMITIVOS
    if (typeTag === "u8") return arr[0];
    if (typeTag === "u16") return new DataView(arr.buffer).getUint16(0, true);
    if (typeTag === "u32") return new DataView(arr.buffer).getUint32(0, true);
    if (typeTag === "u64") {
      // arr viene en formato little-endian → revertir
      const reversed = Array.from(arr).reverse();

      // Convertir a hex WITHOUT Buffer
      const hex = reversed
        .map(b => b.toString(16).padStart(2, "0"))
        .join("");

      // Crear BigInt desde hex
      return BigInt("0x" + hex);
    }


    if (typeTag === "bool") return arr[0] === 1;

    // STRING
    if (typeTag === "0x1::string::String") {
      return decodeBCSString(bytes);
    }

    // VECTOR<STRING>
    if (typeTag.startsWith("vector<0x1::string::String>")) {
      return decodeBCSVectorString(bytes);
    }

    // STRUCT (ej: Nivel)
    if (typeTag.includes("Nivel")) {
      return decodeNivel(bytes);
    }

    return "<?> Tipo no soportado: " + typeTag;
  }

  //
  // ===========================
  // DECODIFICAR STRING BCS
  // ===========================
  //
  function decodeBCSString(bytes) {
    const arr = Uint8Array.from(bytes);
    let length = 0;
    let shift = 0;
    let offset = 0;

    while (offset < arr.length) {
      const byte = arr[offset++];
      length |= (byte & 0x7F) << shift;
      if ((byte & 0x80) === 0) break;
      shift += 7;
    }

    const content = arr.slice(offset, offset + length);
    return new TextDecoder().decode(content);
  }

  //
  // ===========================
  // DECODIFICAR vector<String>
  // ===========================
  //
  function decodeBCSVectorString(bytes) {
    const arr = Uint8Array.from(bytes);
    let offset = 0;

    // tamaño del vector
    let vecLen = 0;
    let shift = 0;

    while (true) {
      const byte = arr[offset++];
      vecLen |= (byte & 0x7F) << shift;
      if ((byte & 0x80) === 0) break;
      shift += 7;
    }

    const items = [];

    for (let i = 0; i < vecLen; i++) {
      // longitud del string
      let len = 0;
      shift = 0;

      while (true) {
        const byte = arr[offset++];
        len |= (byte & 0x7F) << shift;
        if ((byte & 0x80) === 0) break;
        shift += 7;
      }

      const content = arr.slice(offset, offset + len);
      offset += len;

      items.push(new TextDecoder().decode(content));
    }

    return items;
    }

    //
    // ===========================
    // DECODIFICADOR STRUCT NIVEL
    // ===========================
    //
    function decodeNivel(bytes) {
      const arr = Uint8Array.from(bytes);

      if (arr.length < 2) {
        return { tipo: "desconocido", descuento: null };
      }

      const variant = arr[0];
      const descuento = arr[1]; // u8 directo

      const variants = ["cobre", "plata", "oro", "diamante"];

      return {
        tipo: variants[variant] ?? "desconocido",
        descuento
      };
  }
```
La función ClientCall actúa como la capa de comunicación entre el frontend y los contratos Move desplegados en la blockchain de Sui. Su propósito es permitir que el frontend pueda ejecutar cualquier función Move —ya sea mutante o de solo lectura— sin necesidad de preocuparse por formatos, serialización, o detalles internos del BCS.

En ClientCall abstrae toda la complejidad de Sui para que el resto del frontend pueda trabajar con funciones Move como si fueran llamadas JavaScript normales.

## 1. Serialización automática de argumentos

Move requiere que todos los argumentos enviados en un move_call estén correctamente codificados en formato BCS. ClientCall se encarga de convertir cualquier valor recibido desde el frontend en el tipo adecuado:
* ObjectIDs → tx.object(id)
* Strings → tx.pure.string
* Booleans → tx.pure.bool
* Números → tx.pure.u64 o el tipo explícito indicado
* Tipos declarados manualmente → { type, value }

## 2. Construcción del Move Call

Una vez serializados los argumentos, ClientCall construye el bloque de transacción:

```js
tx.moveCall({
  target: `${packageId}::${modulo}::${params.funcion}`,
  arguments: args,
});
```

## 3. Detección de funciones “solo lectura”
Muchas funciones Move solo leen información y no modifican el estado; para este tipo de funciones no se necesita firmar transacciones ni pagar gas. ClientCall detecta si la función es de solo lectura mediante:

Si la función es view:

* Se ejecuta mediante devInspectTransactionBlock
* No se pide firma al usuario
* No se registra en la blockchain
* Se obtiene el valor retornado directamente

## 4. Ejecución de transacciones mutantes

Si la función sí modifica el estado, entonces ClientCall:

* Pide a la wallet que firme la transacción
* La envía a la red
* Espera a que se confirme
* Decodifica los valores retornados, si los hay
* Actualiza estados internos (por ejemplo, nuevos ObjectIDs creados)
  
Este flujo cubre todo el ciclo necesario para operaciones reales en Sui, como crear objetos, actualizarlos o eliminarlos.

## 5. Decodificación de valores retornados (BCS → JavaScript)

Sui devuelve todos los valores en formato BCS, incluso los strings. Por ello, ClientCall incluye un sistema avanzado de decodificación capaz de interpretar:

* u8, u16, u32, u64
* bool
* String
* vector<String>
* enums y structs (como tu enum Nivel)
* tuplas de múltiples valores
