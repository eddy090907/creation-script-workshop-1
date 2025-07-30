# Consultas SQL para Base de Datos Bancaria

## Enunciado 1: Clientes con múltiples cuentas y sus saldos totales

**Necesidad:** El banco necesita identificar a los clientes que tienen más de una cuenta, mostrar cuántas cuentas tiene cada uno y el saldo total acumulado en todas sus cuentas, ordenados por saldo total de mayor a menor.

**Consulta SQL:**
```sql
select cl.nombre, count(ct.num_cuenta) Cuentas, SUM(ct.saldo) Saldo
from cliente cl
inner join cuenta ct on cl.id_cliente = ct.id_cliente 
group by cl.nombre
order by saldo desc;
```

## Enunciado 2: Comparativa entre depósitos y retiros por cliente

**Necesidad:** El departamento de análisis financiero necesita comparar los montos totales de depósitos y retiros realizados por cada cliente, para identificar patrones de comportamiento financiero.

**Consulta SQL:**
```sql
select
    cl.id_cliente,
    cl.nombre,
    coalesce(sum(case when tr.tipo_transaccion = 'deposito' then tr.monto else 0 end), 0) as total_depositos,
    coalesce(sum(case when tr.tipo_transaccion = 'retiro' then tr.monto else 0 end), 0) as total_retiros
from cliente cl
inner join cuenta ct on cl.id_cliente = ct.id_cliente
left join transaccion tr on ct.num_cuenta  = tr.num_cuenta
group by cl.id_cliente, cl.nombre
order by cl.id_cliente;
```

## Enunciado 3: Cuentas sin tarjetas asociadas

**Necesidad:** El departamento de tarjetas necesita identificar todas las cuentas que no tienen tarjetas asociadas para ofrecer productos específicos a estos clientes.

**Consulta SQL:**
```sql
select
	cl.id_cliente,
	cl.nombre,
	ct.num_cuenta 
from cliente cl
inner join cuenta ct on cl.id_cliente = ct.id_cliente
where ct.num_cuenta not in (
	select tj.num_cuenta from tarjeta tj 
);
```

## Enunciado 4: Análisis de saldos promedio por tipo de cuenta y comportamiento transaccional

**Necesidad:** La gerencia necesita un análisis comparativo del saldo promedio entre cuentas de ahorro y corriente, pero solo considerando aquellas cuentas que han tenido al menos una transacción en los últimos 30 días.

**Consulta SQL:**
```sql
with cuentas_con_movimiento as (
    select distinct tx.num_cuenta
    from transaccion tx
    where tx.fecha >= CURRENT_DATE - INTERVAL '30 days'
),
saldo_por_cuenta as (
    select
        ct.num_cuenta,
        ct.tipo_cuenta,
        sum(case when ct.tipo_cuenta = 'ahorro' then tr.monto
                 when ct.tipo_cuenta  = 'corriente' then tr.monto end) as saldo
    from cuenta ct
    inner join cuentas_con_movimiento cm on ct.num_cuenta = cm.num_cuenta
    inner join transaccion tr on ct.num_cuenta = tr.num_cuenta
    group by ct.num_cuenta , ct.tipo_cuenta
)
select
    tipo_cuenta,
    AVG(saldo)::numeric(12,2) as saldo_promedio
from saldo_por_cuenta
group by tipo_cuenta;
```

## Enunciado 5: Clientes con transferencias pero sin retiros en cajeros

**Necesidad:** El equipo de marketing digital necesita identificar a los clientes que utilizan transferencias pero no realizan retiros por cajeros automáticos, para dirigir campañas de banca digital.

**Consulta SQL:**
```sql
select distinct cl.id_cliente, cl.nombre
from cliente cl
inner join cuenta ct on cl.id_cliente = ct.id_cliente
where exists (
    select 1
    from transaccion tx
    where tx.num_cuenta = ct.num_cuenta
      and tx.tipo_transaccion = 'transferencia'
)
and not exists (
    select 1
    from transaccion tr
    where tr.num_cuenta = ct.num_cuenta
    	and tr.tipo_transaccion = 'retiro'
      	and tr.descripcion in ('Retiro en cajero', 'Retiro en cajero automático')
);
```
