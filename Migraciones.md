## Instrucciones
Para las migraciones se debe deben ejecutar paso a paso los script de elastic search, tal cual en el orden en el que estan presentes en este documento.


- 1) Vaciar campo logs_estados para comenzar las migraciones
- 2) Actualizar pagado
- 3) No pagada
- 4) Archivada
- 5) Cargado
- 6) Consultado

## Detalles
- Migracion de estados de las facturas de recaudo para el cliente 11152 y 12382 de Movistar
- Migraciones del mes de Enero y Febrero

### Vaciar campo logs_estados para comenzar las migraciones

```json
PUT recaudo_facturas/_mapping
{
  "script": {
    "lang": "painless",
    "source": "ctx._source.logs_estados = [params.log]",
    "params": {
      "log": []
    }
  },
  "query": {
        "bool": {
            "must": [
                {
                    "terms": {
                        "id_cliente": [
                            "11152",
                            "12382"
                        ]
                    }
                },
                {
                    "range": {
                        "fecha_real": {
                            "gte": "2022-01-01T00:00:00-05:00",
                            "lte": "2022-02-28T23:59:59-05:00"
                        }
                    }
                }
            ]
        }
    }
}
```

### Actualizar pagado

```json
POST recaudo_facturas/_update_by_query
{
  "script": {
    "lang": "painless",
    "source": "for(pago in ctx._source.pagos) { if (pago.estado == 'Aceptada') { params.log.fecha_actualizacion = Instant.ofEpochSecond(pago.fechapago).atZone(ZoneOffset.ofHours(-5)); } } ctx._source.logs_estados = [params.log]",
    "params": {
      "log": {
        "estado": "Pagado",
        "activo": true
      }
    }
  },
  "query": {
    "bool": {
      "must": [
        {
                    "terms": {
                        "id_cliente": [
                            "11152",
                            "12382"
                        ]
                    }
                },
                {
                    "range": {
                        "fecha_real": {
                            "gte": "2022-01-01T00:00:00-05:00",
                            "lte": "2022-02-28T23:59:59-05:00"
                        }
                    }
                },
        {
          "term": {
            "pagos.estado.keyword": {
              "value": "Aceptada"
            }
          }
        }
      ],
      "must_not": [
        {
          "nested": {
            "path": "logs_estados",
            "query": {
              "exists": {
                "field": "logs_estados.estado"
              }
            }
          }
        }
      ]
    }
  }
}
```

```json
POST recaudo_facturas/_update_by_query
{
  "script": {
    "lang": "painless",
    "source": "if(ctx._source.containsKey('fecha') && ctx._source.fecha != null){params.log.fecha_actualizacion = Instant.ofEpochSecond(ctx._source.fecha).atZone(ZoneOffset.ofHours(-5));}else if(ctx._source.containsKey('@timestamp') && ctx._source['@timestamp'] != null){params.log.fecha_actualizacion = ctx._source['@timestamp'].value;}else{params.log.fecha_actualizacion = new Date();} ctx._source.logs_estados = [params.log]",
    "params": {
      "log": {
        "estado": "Pagado",
        "activo": true
      }
    }
  },
  "query": {
    "bool": {
      "must": [
          {
                    "terms": {
                        "id_cliente": [
                            "11152",
                            "12382"
                        ]
                    }
                },
                {
                    "range": {
                        "fecha_real": {
                            "gte": "2022-01-01T00:00:00-05:00",
                            "lte": "2022-02-28T23:59:59-05:00"
                        }
                    }
                },
        {
          "term": {
            "estado_pago.keyword": {
              "value": "Aceptada"
            }
          }
        }
      ],
      "must_not": [
        {
          "nested": {
            "path": "logs_estados",
            "query": {
              "exists": {
                "field": "logs_estados.estado"
              }
            }
          }
        }
      ]
    }
  }
}
```

### No pagada

```json
POST recaudo_facturas/_update_by_query
{
  "script": {
    "lang": "painless",
    "source": "if(ctx._source.containsKey('fecha') && ctx._source.fecha != null){params.log.fecha_actualizacion = Instant.ofEpochSecond(ctx._source.fecha).atZone(ZoneOffset.ofHours(-5));}else if(ctx._source.containsKey('@timestamp') && ctx._source['@timestamp'] != null){params.log.fecha_actualizacion = ctx._source['@timestamp'].value;}else{params.log.fecha_actualizacion = new Date();} ctx._source.logs_estados = [params.log]",
    "params": {
      "log": {
        "estado": "No Pagado",
        "activo": true
      }
    }
  },
  "query": {
    "bool": {
      "must": [
          {
                    "terms": {
                        "id_cliente": [
                            "11152",
                            "12382"
                        ]
                    }
                },
                {
                    "range": {
                        "fecha_real": {
                            "gte": "2022-01-01T00:00:00-05:00",
                            "lte": "2022-02-28T23:59:59-05:00"
                        }
                    }
                },
        {
          "term": {
            "tipo": {
              "value": "web"
            }
          }
        }
      ],
      "must_not": [
        {
          "nested": {
            "path": "logs_estados",
            "query": {
              "exists": {
                "field": "logs_estados.estado"
              }
            }
          }
        },
        {
          "exists": {
            "field": "pagos"
          }
        }
      ]
    }
  }
}
```

```json
POST recaudo_facturas/_update_by_query
{
  "script": {
    "lang": "painless",
    "source": "for(pago in ctx._source.pagos) { if (pago.estado != 'Aceptada') { params.log.fecha_actualizacion = Instant.ofEpochSecond(pago.fechapago).atZone(ZoneOffset.ofHours(-5)); } } ctx._source.logs_estados = [params.log]",
    "params": {
      "log": {
        "estado": "No Pagado",
        "activo": true
      }
    }
  },
   "query": {
    "bool": {
      "must": [
          {
                    "terms": {
                        "id_cliente": [
                            "11152",
                            "12382"
                        ]
                    }
                },
                {
                    "range": {
                        "fecha_real": {
                            "gte": "2022-01-01T00:00:00-05:00",
                            "lte": "2022-02-28T23:59:59-05:00"
                        }
                    }
                },
        {
          "exists": {
            "field": "pagos"
          }
        }
      ],
      "must_not": [
        {
          "nested": {
            "path": "logs_estados",
            "query": {
              "exists": {
                "field": "logs_estados.estado"
              }
            }
          }
        },
        {
          "term": {
            "pagos.estado.keyword": {
              "value": "Aceptada"
            }
          }
        }
      ]
    }
  }
}
```

### Archivada

```json
POST recaudo_facturas/_update_by_query
{
  "script": {
    "lang": "painless",
    "source": "params.log.fecha_actualizacion =  new Date(); ctx._source.logs_estados = [params.log]",
    "params": {
      "log": {
        "estado": "Archivado",
        "activo": true
      }
    }
  },
  "query": {
    "bool": {
      "must": [
          {
                    "terms": {
                        "id_cliente": [
                            "11152",
                            "12382"
                        ]
                    }
                },
                {
                    "range": {
                        "fecha_real": {
                            "gte": "2022-01-01T00:00:00-05:00",
                            "lte": "2022-02-28T23:59:59-05:00"
                        }
                    }
                },
        {
          "term": {
            "tipo.keyword": {
              "value": "archivo"
            }
          }
        },
        {
          "term": {
            "borrado": {
              "value": true
            }
          }
        }
      ],
      "must_not": [
        {
          "nested": {
            "path": "logs_estados",
            "query": {
              "exists": {
                "field": "logs_estados.estado"
              }
            }
          }
        }
      ]
    }
  }
}
```

### Cargado

```json
POST recaudo_facturas/_update_by_query
{
  "script": {
    "lang": "painless",
    "source": "params.log.fecha_actualizacion = Instant.ofEpochSecond(ctx._source.fecha).atZone(ZoneOffset.ofHours(-5)); ctx._source.logs_estados = [params.log]",
    "params": {
      "log": {
        "estado": "Cargado",
        "activo": true
      }
    }
  },
   "query": {
    "bool": {
      "must": [
          {
                    "terms": {
                        "id_cliente": [
                            "11152",
                            "12382"
                        ]
                    }
                },
                {
                    "range": {
                        "fecha_real": {
                            "gte": "2022-01-01T00:00:00-05:00",
                            "lte": "2022-02-28T23:59:59-05:00"
                        }
                    }
                },
        {
          "term": {
            "tipo.keyword": {
              "value": "archivo"
            }
          }
        },
        {
          "term": {
            "borrado": {
              "value": false
            }
          }
        }
      ],
      "must_not": [
        {
          "nested": {
            "path": "logs_estados",
            "query": {
              "exists": {
                "field": "logs_estados.estado"
              }
            }
          }
        },
        {
          "exists": {
            "field": "pagos"
          }
        }
      ]
    }
  }
}
```

### Consultado

```json
POST recaudo_facturas/_update_by_query
{
  "script": {
    "lang": "painless",
    "source": "if(ctx._source.containsKey('fecha') && ctx._source.fecha != null){params.log.fecha_actualizacion = Instant.ofEpochSecond(ctx._source.fecha).atZone(ZoneOffset.ofHours(-5));}else if(ctx._source.containsKey('@timestamp') && ctx._source['@timestamp'] != null){params.log.fecha_actualizacion = ctx._source['@timestamp'].value;}else{params.log.fecha_actualizacion = new Date();} ctx._source.logs_estados = [params.log]",
    "params": {
      "log": {
        "estado": "Consultado",
        "activo": true
      }
    }
  },
  "query": {
    "bool": {
      "must": [
          {
                    "terms": {
                        "id_cliente": [
                            "11152",
                            "12382"
                        ]
                    }
                },
                {
                    "range": {
                        "fecha_real": {
                            "gte": "2022-01-01T00:00:00-05:00",
                            "lte": "2022-02-28T23:59:59-05:00"
                        }
                    }
                },
        {
          "term": {
            "tipo": {
              "value": "online"
            }
          }
        }
      ],
      "must_not": [
        {
          "nested": {
            "path": "logs_estados",
            "query": {
              "exists": {
                "field": "logs_estados.estado"
              }
            }
          }
        },
        {
          "exists": {
            "field": "pagos"
          }
        },
        {
          "exists": {
            "field": "borrado"
          }
        }
      ]
    }
  }
}
```
