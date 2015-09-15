---
layout: post
title:  "Buscando vendedores similares"
date:   2015-09-12 11:45:24
categories: machine_learning python MercadoLibre
---


MercadoLibre es un marketplace para la compra/venta de todo tipo de productos. A través del sitio http://developers.mercadolibre.com/ cualquier desarrollador tiene la posibilidad de acceder a una gran cantidad de datos de cada transacción que tiene lugar en la plataforma.

Un caso que me resulta interesante analizar es, dado un vendedor, buscar(y encontrar) vendedores similares. Cuando vendemos productos en pocas categorías puede ser relativamente sencillo saber cuales son nuestros principales competidores, pero si tenemos una amplia gama de productos en muchas categorías diferentes, esta tarea puede resultar bastante complicada.

Existen muchas variables para identificar vendedores similares a partir de los datos provistos por el API de MercadoLibre:

Tipo de usuario
Reputación
Categorías en las que participan
Fecha de inicio de actividades
Cantidad de ventas diarias/semanales/mensuales
Ingresos por ventas diarias/semanales/mensuales
Este artículo se va a concentrar en la combinación de dos variables: categorías en las que participan e ingresos por ventas en los últimos 30 días. Por lo tanto, para dejarlo en claro, se analizará una matriz en la que cada fila será un vendedor y cada columna una categoría, cada celda tendrá el total de ingresos por ventas de ese vendedor en esa categoría. MercadoLibre tiene una gran cantidad de categorías, lo que va a significar que la cantidad de columnas de la matriz va a ser muy grande y la matriz claramente va a resultar del tipo "sparse" o poco densa, ya que cada vendedor vende productos en muy pocas de esas categorías y por lo tanto la mayor parte de las celdas tendrá un valor nulo(NA).

Para dejarlo más claro este sería un ejemplo:

VENDEDOR(ID)	CATEGORIA A	CATEGORIA B
4312221	551.0	0.0
66655355	NA	0.0
21133313	333.0	5900.0
Para encontrar las filas(vendedores) más similares a la seleccionada hay muchas formas de llegar a la solución. En este artículo voy a utilizar un algoritmo de aprendizaje (relacionado a Machine Learning) llamado Nearest Neighbors(NN), en este caso la variante no supervisada.

El algoritmo NN se encarga de buscar los k-NN(los k vecinos más cercanos) del elemento que querramos analizar comparando cuan cerca está un elemento de otro. Quizá se hayan dado cuenta que este problema se puede resolver tranquilamente calculando una matriz de distancias utilizando una métrica sencilla como la euclideana y estarían completamente en lo cierto, pero hay muchos casos donde la cantidad de datos es demasiado grande y ese cálculo donde se compara una fila con las otras N-1 filas resulta muy "caro" de procesar. Si utilizamos este enfoque(fuerza bruta) y tenemos V vendedores y C categorías la complejidad crece $latex O[CV^2]$.

En la implementación de la excelente librería de Machine Learning para Python llamada Scikit-Learn, además del enfoque de fuerza bruta se pueden utilizar otros dos: KDtree y Ball-tree. El razonamiento básicamente es ahorrar comparaciones entre filas: si la fila A tiene valores muy distantes a la fila B, y la fila B está muy cerca de la C, entonces se puede deducir que las filas A y C están muy distantes. El costo computacional se puede reducir a $latex O[C*V*log(V)]$ o menos, lo que supone una mejora importante.

En cuanto a las métricas de distancia también hay varias opciones, pero en este caso optaré por la euclideana y en el código se va a ver que diferencia hay con respecto a calcular directamente la distancia utilizando esa métrica.

Para el código se utilizarán las librerías pandas, scipy y sklearn:

Importo las librerías necesarias:

{% highlight python %}
import pandas as pd
from scipy.spatial.distance import pdist, squareform
from sklearn.neighbors import NearestNeighbors
{% endhighlight %}
Para leer y procesar el csv que contiene los datos utilizo la librería pandas.

{% highlight python %}
df = pd.read_csv('sellers30-1000.csv', index_col=0)
df2 = df.fillna(0)
x = df2
X = x.values
{% endhighlight %}
Una vez tenemos la matrix de valores X, ya se está en condiciones de usar el algoritmo de Nearest Neighbors. Además se calcula la matriz de distancia directamente, utilizando la métrica "euclidean":

{% highlight python %}
nbrs = NearestNeighbors(n_neighbors=11, algorithm='ball_tree').fit(X)
distances, indices = nbrs.kneighbors(X)
euc_dist = squareform(pdist(X, 'euclidean'))
{% endhighlight %}
El resultado siguiente demuestra que la matriz de distancias creada a través de NN y la creada a través de "euclidean" son prácticamente iguales.

{% highlight python %}
distances[36]
array([    0.  ,   228.73,   231.27,  1231.73,  1478.73,  1618.73,
    1823.73,  2758.73,  2891.27,  2999.81,  4278.78])

euc_dist[36].argsort()[1:11]
array([ 953, 1396,  850,   43, 1330,  491, 1579,  841, 2002, 2270])

indices[36][1:11]
array([ 953, 1396,  850,   43, 1330,  491, 1579,  841, 2002, 2270])
{% endhighlight %}
Finalmente vamos a probar los resultados analizando dos casos, en el primero se puede ver que hay similaridades en los tres vendedores ya que estos se dedican a vender insumos, entre otras cosas.

{% highlight python %}
x.iloc[[36]]
#http://listado.mercadolibre.com.ar/_CustId_60353686
x.iloc[[953]]
#http://listado.mercadolibre.com.ar/_CustId_91988078
x.iloc[[1396]]
#http://eshops.mercadolibre.com.ar/bairesinsumos
{% endhighlight %}
En el segundo caso estos vendedores se dedican principalmente a la ventas de videojuegos para consolas:

{% highlight python %}
x.iloc[[47]]
#http://listado.mercadolibre.com.ar/_CustId_173531278
x.iloc[[1245]]
#http://listado.mercadolibre.com.ar/_CustId_63991948
x.iloc[[1089]]
#http://listado.mercadolibre.com.ar/_CustId_149556862
{% endhighlight %}
Si desean pueden entrar a los links comentados en el código para ver los productos a la venta de cada vendedor.

En próximos artículos se van a explorar otros enfoques y métricas relacionados al mundo de las finanzas como la Affinity Matrix.

 
