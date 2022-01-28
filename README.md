# Introducción

Aquí tienes el enunciado del modulo 2, creat un repo en Github, y añade un readme.md
incluyendo enunciado y consulya (lo que pone aquí _Pega aquí tu consulta_)

# Basico

## CRUD

Crear una BBDD y hacer CRUD

## Restaurar backup

Vamos a restaurar el set de datos de mongo atlas _airbnb_.

Lo puedes encontrar en este enlace: https://drive.google.com/drive/folders/1gAtZZdrBKiKioJSZwnShXskaKk6H_gCJ?usp=sharing

Para restaurarlo puede seguir las instrucciones de este videopost:
https://www.lemoncode.tv/curso/docker-y-mongodb/leccion/restaurando-backup-mongodb

> Acuerdate de mirar si en opt/app hay contenido de backups previos que tengas
> que borrar

## General

En este base de datos puedes encontrar un montón de apartementos y sus
reviews, esto está sacado de hacer webscrapping.

**Pregunta** Si montarás un sitio real, ¿Qué posible problemas pontenciales
les ves a como está almacenada la información?

```md
Incluye todos los comentarios asociados, podiendo hacer que el documento sea muy pesado y supere el límite de los 16MB.

```

## Consultas

### Basico

- Saca en una consulta cuantos apartamentos hay en España.

```js
use('airbnb')
db.listingsAndReviews.count(
  {"address.country_code": "ES"}  
)
```

- Lista los 10 primeros:

  - Sólo muestra: nombre, camas, precio, government_area
  - Ordenados por precio.

```js
use('airbnb')  
db.listingsAndReviews.find(   
  {} ,
    {name:1, beds:1, price:1, "address.government_area":1, _id:0}
)
.sort({price:1})
.limit(10)
```

### Filtrando

- Queremos viajar comodos, somos 4 personas y queremos:
  - 4 camas.
  - Dos cuartos de baño.

```js
use('airbnb')  
db.listingsAndReviews.find(   
  {beds:4, bathrooms: 2} ,
  {name:1, beds:1, bathrooms:1, _id:0}
)  
```

- Al requisito anterior,hay que añadir que nos gusta la tecnología
  queremos que el apartamento tenga wifi.

```js
use('airbnb')  
db.listingsAndReviews.find(   
  {
    beds:4, 
    bathrooms: 2, 
    amenities: {$in: ["Wifi"]}
  },
  {name:1, beds:1, bathrooms:1, amenities: 1, _id:0}
)  
```

- Y bueno, un amigo se ha unido que trae un perro, así que a la query anterior tenemos que
  buscar que permitan mascota _Pets Allowed_

```js
use('airbnb')  
db.listingsAndReviews.find(   
  {
    beds:4, 
    bathrooms: 2, 
    amenities: {$all: ["Pets allowed", "Wifi"]}
  },
  {name:1, beds:1, bathrooms:1, amenities: 1, _id:0}
) 
```

### Operadores lógicos

- Estamos entre ir a Barcelona o a Portugal, los dos destinos nos valen, peeero... queremos que
  el precio nos salga baratito (50 $), y que tenga buen rating de reviews

```js
use('airbnb')  
db.listingsAndReviews.find(   
  {
    price: {$lte: 50},
    "review_scores.review_scores_rating": {$gte: 85},
    $or: [
      {"address.country": "Portugal"},
      {"address.market": "Barcelona"}
    ]
  },
  {name:1, price:1, "address.market":1, "address.country": 1, "review_scores.review_scores_rating": 1, _id:0}
)  
```

## Agregaciones

# Basico

- Queremos mostrar los pisos que hay en España, y los siguiente campos:
  - Nombre.
  - De que ciudad (no queremos mostrar un objeto, sólo el string con la ciudad)
  - El precio (no queremos mostrar un objeto, sólo el campo de precio)

```js
[
  {
    $match: {
      'address.country': 'Spain'
    }
  }, 
  {
    $project: {      
      nombre: "$name",
      ciudad: "$address.market",
      precio: "$price",
      _id: 0
    }
  }
]
```

- Queremos saber cuantos alojamientos hay disponibles por pais.

```js
[
  {
    $group: {
      _id: '$address.country',
      alojamientos: {
          $sum: 1
      }
    }
  }, 
  {
    $project: {
      pais: '$_id',
      alojamientos: 1,
      _id: 0
    }
  }, 
  {
    $sort: {
      pais: 1
    }
  }
]
```

# Opcional

- Queremos saber el precio medio de alquiler de airbnb en España.

```js
[
  {
    $match: {
      'address.country': 'Spain'
    }
  }, {
    $group: {
      _id: '$address.country',
      precio_medio: {
        $avg: '$price'
      }
    }
  }, {
    $project: {
      pais: '$_id',
      precio_medio: 1,
      _id: 0
    }
  }
]
```

- ¿Y si quisieramos hacer como el anterior, pero sacarlo por paises?

```js
[
  {
    $group: {
      _id: '$address.country',
      precio_medio: {
        $avg: '$price'
      }
    }
  }, {
    $sort: {
      _id: 1
    }
  }, {
    $project: {
      pais: '$_id',
      precio_medio: 1,
      _id: 0
    }
  }
]
```

- Repite los mismos pasos pero agrupando también por numero de habitaciones.

```js
[
  {
    $group: {
      _id: {
        pais: '$address.country',
        camas: '$beds'
      },
      precio_medio: {
        $avg: '$price'
      }
    }
    }, {
    $sort: {
      '_id.pais': 1,
      '_id.camas': 1
    }
    }, {
    $project: {
      pais: '$_id.pais',
      camas: '$_id.camas',
      precio_medio: 1,
      _id: 0
    }
  }
]
```

# Desafio

Queremos mostrar el top 5 de apartamentos más caros en España, y sacar
los siguentes campos:

- Nombre.
- Ciudad.
- Amenities, pero en vez de un array, un string con todos los ammenities.

```js
[
  {
    $match: {
      'address.country': 'Spain'
    }
  }, {
    $sort: {
      price: -1
    }
  }, {
    $limit: 5
  }, {
    $addFields: {
      amenitiesList: {
        $reduce: {
          input: '$amenities',
          initialValue: '',
          'in': {
            $concat: [
              '$$value',
              ', ',
              '$$this'
            ]
          }
        }
      }
    }
  }, {
    $project: {
      Nombre: '$name',
      Ciudad: '$address.market',
      Precio: '$price',
      lista_amenities: {
        $substr: [
          '$amenitiesList',
          2,
          -1
        ]
      },
      _id: 0
    }
  }
]
```