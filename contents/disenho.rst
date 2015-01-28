==========================
Desarrollo. Diseño
==========================

Diseño normalizado (favorecemos escrituras) vs Diseño no normalizado (favorecemos lecturas pero hay duplicados).

A la hora de diseñar con mongo se suele tirar hacia el lado del diseño no normalizado.

.. note::
    Hay que intentar siempre minimizar la lista de documentos que se devuelven al driver

------------

*Ejemplo:*

Queremos almacenar un conteo de los accesos realizados sobre un recurso. Hay que tenerlos controlados por día, hora y minuto

Consultas típicas: información de conteos, recursos mas vistos...

Elegir posibles candidatos a shardkey.

* sample: ip, http_method, http_code, fecha, ...
* recurso: /index.htm, /productos...
* hit -> contador

Una opción sería: ::

    {_id, ip, http_method, http_code, fecha, fecha_completa, recurso}
    shardKey = recurso, fecha

    Con una agregación realizar crear la nueva colección que se usará en las consultas típicas
    { id={dia, hora, minuto}, hits}


--------------

*Ejemplo:*

Dados los datos de los empleados: información básica: nif, nombre, ...

Queremos obtener informes de gastos por empleado, mes y año, de la forma:

* gasto

  * proyecto

    * tiempo
    * concepto
    * importe

  * total
  * estado (nuevo, emitido, confirmado, pagado, denegado)

Para confirmado, pagado o denegado hay una serie de empleados (técnicos o lo que sea) que pueden modificar el estado del gasto

.. code-block :: json

    empleados:
    {
      id:
      nombre_empleado:
      inf_basica:
      supervisa: [<ids empleados>]
    }

    gastos:
    {
      _id: {proyecto: "proyecto", anho:..., mes:...}
      gastos : [
        {
          _id:
          id_empleado:
          tiempo:
          concepto:
          gasto:
          estado:
        }
      ]
    }

