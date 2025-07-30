# Consultas NoSQL para Sistema Bancario en MongoDB

A continuación se presentan 5 enunciados de consultas basados en las colecciones NoSQL del sistema bancario, junto con las soluciones utilizando operaciones avanzadas de MongoDB.

## 1. Análisis de Saldos por Tipo de Cuenta

**Enunciado:** El departamento financiero necesita un informe que muestre el saldo total, promedio, máximo y mínimo por cada tipo de cuenta (ahorro y corriente) para evaluar la distribución de fondos en el banco.

**Consulta MongoDB:**
```javascript
db.clientes.aggregate([
  {
    $unwind: "$cuentas"
  },
  {
    $group: {
      _id: "$cuentas.tipo_cuenta",
      totalSaldo: { $sum: "$cuentas.saldo" },
      promedioSaldo: { $avg: "$cuentas.saldo" },
      saldoMaximo: { $max: "$cuentas.saldo" },
      saldoMinimo: { $min: "$cuentas.saldo" }
    }
  },
  {
    $project: {
      _id: 0,
      tipoCuenta: "$_id",
      totalSaldo: 1,
      promedioSaldo: 1,
      saldoMaximo: 1,
      saldoMinimo: 1
    }
  }
])
```

## 2. Patrones de Transacciones por Cliente

**Enunciado:** El equipo de análisis de comportamiento necesita identificar los patrones de transacciones de cada cliente, mostrando la cantidad y el monto total de transacciones por tipo (depósito, retiro, transferencia) para cada cliente.

**Consulta MongoDB:**
```javascript
db.transacciones.aggregate([
  {
    $group: {
      _id: {
        cliente: "$cliente_ref",
        tipo: "$tipo_transaccion"
      },
      cantidadTransacciones: { $sum: 1 },
      montoTotal: { $sum: "$monto" }
    }
  },
  {
    $group: {
      _id: "$_id.cliente",
      transaccionesPorTipo: {
        $push: {
          tipoTransaccion: "$_id.tipo",
          cantidad: "$cantidadTransacciones",
          total: "$montoTotal"
        }
      }
    }
  },
  {
    $project: {
      _id: 0,
      cliente: "$_id",
      transaccionesPorTipo: 1
    }
  }
])
```

## 3. Clientes con Múltiples Tarjetas de Crédito

**Enunciado:** El departamento de riesgo crediticio necesita identificar a los clientes que poseen más de una tarjeta de crédito, mostrando sus datos personales, cantidad de tarjetas y el detalle de cada una.

**Consulta MongoDB:**
```javascript
db.clientes.aggregate([
  // Separar cuentas
  { $unwind: "$cuentas" },
  // Separar tarjetas dentro de cada cuenta
  { $unwind: "$cuentas.tarjetas" },
  // Filtrar solo tarjetas de crédito
  {
    $match: {
      "cuentas.tarjetas.tipo_tarjeta": "credito"
    }
  },
  // Agrupar por cliente para contar y recolectar detalles
  {
    $group: {
      _id: "$_id",
      nombre: { $first: "$nombre" },
      cedula: { $first: "$cedula" },
      correo: { $first: "$correo" },
      direccion: { $first: "$direccion" },
      cantidadTarjetasCredito: { $sum: 1 },
      tarjetasCredito: { $push: "$cuentas.tarjetas" }
    }
  },
  // Filtrar clientes con más de una tarjeta de crédito
  {
    $match: {
      cantidadTarjetasCredito: { $gt: 1 }
    }
  },
  // Dar formato a la salida
  {
    $project: {
      _id: 0,
      nombre: 1,
      cedula: 1,
      correo: 1,
      direccion: 1,
      cantidadTarjetasCredito: 1,
      tarjetasCredito: 1
    }
  }
])
```

## 4. Análisis de Medios de Pago más Utilizados

**Enunciado:** El departamento de marketing necesita conocer cuáles son los medios de pago más utilizados para depósitos, agrupados por mes, para orientar sus campañas promocionales.

**Consulta MongoDB:**
```javascript
db.transacciones.aggregate([
  // Filtrar solo transacciones de tipo depósito
  {
    $match: {
      tipo_transaccion: "deposito"
    }
  },
  // Crear campo 'mes' con formato YYYY-MM
  {
    $addFields: {
      mes: {
        $dateToString: {
          format: "%Y-%m",
          date: { $toDate: "$fecha" }
        }
      }
    }
  },
  // Se agrupa por mes y medio de pago
  {
    $group: {
      _id: {
        mes: "$mes",
        medioPago: "$detalles_deposito.medio_pago"
      },
      cantidad: { $sum: 1 }
    }
  },
  // Se da formato a la salida
  {
    $project: {
      _id: 0,
      mes: "$_id.mes",
      medioPago: "$_id.medioPago",
      cantidad: 1
    }
  },
  // Se ordenar por mes y cantidad descendente
  {
    $sort: {
      mes: 1,
      cantidad: -1
    }
  }
])
```

## 5. Detección de Cuentas con Transacciones Sospechosas

**Enunciado:** El departamento de seguridad necesita identificar cuentas con patrones de transacciones sospechosas, definidas como aquellas que tienen más de 3 retiros en un mismo día con un monto total superior a 1,000,000 COP.

**Consulta MongoDB:**
```javascript
db.transacciones.aggregate([
  // Filtrar solo retiros
  {
    $match: {
      tipo_transaccion: "retiro"
    }
  },
  // Extraer solo la fecha sin hora (ej: 2023-05-15)
  {
    $addFields: {
      fechaDia: {
        $dateToString: {
          format: "%Y-%m-%d",
          date: { $toDate: "$fecha" }
        }
      }
    }
  },
  // Agrupar por cuenta y fecha del día
  {
    $group: {
      _id: {
        numCuenta: "$num_cuenta",
        fechaDia: "$fechaDia"
      },
      cantidadRetiros: { $sum: 1 },
      montoTotal: { $sum: "$monto" },
      detalles: { $push: "$$ROOT" }
    }
  },
  // Filtrar los patrones sospechosos
  {
    $match: {
      cantidadRetiros: { $gt: 0 },
      montoTotal: { $gt: 1000000 }
    }
  },
  // Formatear la salida
  {
    $project: {
      _id: 0,
      numCuenta: "$_id.numCuenta",
      fecha: "$_id.fechaDia",
      cantidadRetiros: 1,
      montoTotal: 1,
      detalles: 1
    }
  }
])
```
