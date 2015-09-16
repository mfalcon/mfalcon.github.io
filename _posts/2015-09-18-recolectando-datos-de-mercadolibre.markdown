---
layout: post
title:  "Recolectando datos de MercadoLibre"
date:   2015-09-18 14:41:24
categories: python MercadoLibre
---


Como había mencionado en otro post, a través del sitio de [developers](http://developers.mercadolibre.com/) cualquier desarrollador tiene la posibilidad de acceder a una gran cantidad de datos de cada transacción que tiene lugar en la plataforma de MercadoLibre.

Para hacer todos los análisis relacionados tuve que armar una aplicación para consumir los datos de su API de forma diaria, el código está desarrollado en Python,usa PostgreSQL como base de datos y beanstalkd como cola de mensajes. Si quieren ver el código, lo pueden encontrar en [https://github.com/mfalcon/meli-collection](github).
Básicamente hay cuatro archivos:

- meli_api.py: es el encargado de realizar requests al API de MercadoLibre.
- pages_collector.py: Recorre el árbol de categorías de MercadoLibre y las inserta en una cola.
- postgresmeli.py: Extrae páginas de la cola para luego obtener los items que la componen y guardarlos en la base de datos.
- data_updater.py: Actualiza datos de categorías y vendedores.

En este post les quiero mostrar algunas partes que me parecen interesantes de compartir.

Empecemos: 

{% highlight python %}
#postgresmeli.py 
def chunks(l, n):
    """ Yield successive n-sized chunks from l.
    """
    for i in xrange(0, len(l), n):
        yield l[i:i+n]
{% endhighlight %}

La función chunks se encarga de devolver un "pedazo" de determinada lista que pasa como parámetro. El keyword "yield" se encarga de
crear un generador, que luego podrá ser iterado en el código a continuación:

{% highlight python %}
item_ids = [item['id'] for item in items]
item_chunks = chunks(item_ids, ITEMS_IDS_LIMIT)
for item_ids in item_chunks:
    items_data = self.mapi.get_items_data(item_ids)
{% endhighlight %}

La siguiente función se encarga de insertar filas de a bloques para reducir la carga de I/O:

{% highlight python %}
#postgresmeli.py 
def add_row_bulk(self, data, table): #data is a dictionary
    columns = data[0].keys()
    cols = (AsIs(','.join(columns)))
    value_rows = []
    for e in data:
        columns = e.keys()
        values = []
    
        for column in columns:
            if isinstance(e[column], list): #checking for json values
                values.append(Json(e[column])) 
            elif isinstance(e[column], dict):
                values.append(Json(e[column])) 
            else:
                values.append(e[column])
        value_rows.append(tuple(values))        
  
    if table == 'item':
        first_arg = 'INSERT INTO item (%s) VALUES ' + ','.join(['%s'] * len(value_rows))
    elif table == 'item_status':
        first_arg = 'INSERT INTO item_status (%s) VALUES ' + ','.join(['%s'] * len(value_rows))
        
    second_arg = [cols] + value_rows
    query = self.cur.mogrify(first_arg, second_arg)
    self.cur.execute(query) 
    self.conn.commit()
    print "added %s" % table
{% endhighlight %}

Mogrify ....

Ahora pasamos a otro script:

{% highlight python %}
#data_updater.py
def insert_all_categories(self, cats):
    for cat_id in cats:
        def _get_leaf_nodes(cat_id):
            cat_in_db = self.find_one('category', cat_id)
            cat_data = self.mapi.get_category(cat_id)
            if cat_in_db:
                #check_if_changed(cat_in_db, cat_data)
                pass
            else:
                data = {k: cat_data[k] for k in ('id', 'name', 'path_from_root', 'children_categories')}
                self.add_row(data, 'category')
                print "added %s" % cat_data['name']
                
            if len(cat_data['children_categories']) == 0:
                #leaf cat, end of recursion
                print "leaf cat"
            else:
                for cat in cat_data['children_categories']:
                    _get_leaf_nodes(cat['id'])
        
        _get_leaf_nodes(cat_id)
{% endhighlight %}

Mercadolibre cuenta con un árbol jerárquico de categorías. Esta función recorre recursivamente el árbol para insertar todas las categorías
que no se hayan creado anteriormente, la recursión terminará cuando no haya más hijos.


Pasando al script donde se hacen las llamadas al API de MercadoLibre, quiero hacer incapié en el código encargado de manejar
los requests:

{% highlight python %}
#meli_api.py
def make_call(self, url, params=None):
    time.sleep(SLEEP_TIME)
    for i in range(10):
        if i != 0:
            self.logger.info("%s - retrying... %d" % (url,i))
            time.sleep(ERROR_SLEEP_TIME*i)
        try:
            res = requests.get(url, verify=False)
        except requests.ConnectionError, e:
            self.logger.info(e)
            continue
        
        if res.status_code == 200:
            #data = json.loads(res.text)
            data = res.json()
            if data:
                return data
            continue
{% endhighlight %}




 
