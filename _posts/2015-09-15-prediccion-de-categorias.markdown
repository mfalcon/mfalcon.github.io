---
layout: post
title:  "Prediccion de categorias"
date:   2015-09-15 14:45:24
categories: machine_learning python MercadoLibre
---


Hace un tiempo leí un paper del equipo de R&D de la empresa Ebay donde explicaba el proceso de implementación de un algoritmo que permitiría sugerir categorías al momento en que un vendedor esté ingresando el título del producto que desea vender. Me pareció algo interesante, que ahorra tiempo y ayuda a reducir errores humanos, así que miré si MercadoLibre(ML) tenía algo similar pero en ese momento no lo vi(no sé ahora).

Es por eso que decidí escribir un poco de código para desarrollar una solución. Hay muchas alternativas a la hora de afrontar esta solución y en este caso vamos a seleccionar la más sencilla: dado un título predecir a que categoría corresponde entrenando el modelo de aprendizaje únicamente con estas dos variables.

Para lograr el objetivo voy a utilizar un algoritmo de Machine Learning supervisado llamado Linear SVC(Linear Support Vector Classification).  La teoría detrás de este algoritmo es algo compleja y desvía el sentido práctico de este post. Se puede encontrar mucha información sobre el algoritmo en internet, el MOOC de Coursera tiene buen contenido, por ejemplo [aquí](http://cs229.stanford.edu/notes/cs229-notes3.pdf) (es un pdf).

Al ser un algoritmo de ML del tipo supervisado, significa que debemos "entrenar" un modelo con datos 100% confiables en donde se sabe a que "categoría" pertenece cada elemento, este conjunto de datos se denomina "training set". Una vez el modelo ha sido entrenado debemos probar su precisión, esto lo hacemos reservando una porción del training set. Como sabemos en que categoría se ubica cada elemento, podemos esconder ese dato al modelo para luego evaluar y comparar el resultado arrojado por el modelo contra el resultado real, este conjunto de datos es llamado "testing set".

Como las variables de entrada son del tipo texto, lo primero que hay que hacer es pasarlas al mundo numérico. Esto lo haré utilizando la metodología Bag of Words que básicamente arma una matriz de ocurrencias de palabras en cada variable de entrada( o sea en cada fila). La función CountVectorizer() es la encargada de hacer eso en la librería scikit-learn.

{% highlight python %}

count_vect = CountVectorizer()
X_train_counts = count_vect.fit_transform([e[0] for e in training_set])

{% endhighlight %}

El problema con entrenar el modelo con esta matriz es que se le otorgaría mucha relevancia a palabras comunes como en el caso de títulos de items de MercadoLibre pueden ser: nuevo, celular, notebook, garantia, ... Para mejorar la precisión del modelo se utilizará TF-IDF, de esta forma las palabras menos informativas tendrán menor peso a la hora de clasificar documentos.

{% highlight python %}
tfidf_transformer = TfidfTransformer() 
X_train_tfidf = tfidf_transformer.fit_transform(X_train_counts)
X_train = X_train_tfidf
{% endhighlight %}

Ahora sí se puede entrenar al clasificador, C es un parámetro de penalidad que se le pasa al algoritmo, es posible no pasarlo y dejar que tome el default(1.0). En mi caso puse  C=2.28 porque utilicé previamente una técnica de "Parameter Tuning" que explicaré más adelante en un post más teórico.

{% highlight python %}

clf = svm.LinearSVC(C=2.28).fit(X_train, [e[2] for e in training_set])

{% endhighlight %}
Luego de entrenar el modelo, evaluamos su precisión con el test_set:

{% highlight python %}

docs_test = [e[0] for e in test_set]
 X_test_counts = count_vect.transform(docs_test)
 X_test_tfidf = tfidf_transformer.transform(X_test_counts)
 X_test = X_test_tfidf
 predicted = clf.predict(X_test)
 accuracy = np.mean(predicted == [e[2] for e in test_set])
 print "accuracy: %.8f" % accuracy
{% endhighlight %}

Y ahora resta probar con un par de títulos y ver que predice el clasificador:

{% highlight python %}

docs_new = ['notebook bangho i3 4gb','tp link 300mbps', 'skullcdy','armada i3', 'genius mouse usb']
 X_new_counts = count_vect.transform(docs_new)
 X_new_tfidf = tfidf_transformer.transform(X_new_counts)
 predicted = clf.predict(X_new_tfidf)

 for idx, predicted_cat in enumerate(predicted):
 cur.execute(cat_query % predicted_cat)
 data = cur.fetchall()[0][0]
 item_cats = [e['name'] for e in data]
 item_title = docs_new[idx]
 print "%s is probably part of: " % item_title + " > ".join(item_cats)
{% endhighlight %}

Y esta es la respuesta que se obtiene:

{% highlight python %}
accuracy: 0.83066321
notebook bangho i3 4gb is probably part of: Computación > Notebooks y Accesorios > Notebooks > Bangho > Intel Core i3 > 15" a 15,9"
tp link 300mbps is probably part of: Computación > Redes > Redes Inalámbricas - Wireless > Routers y Access Points > Tp-Link
skullcady is probably part of: Computación > Otros
armada i3 is probably part of: Computación > PC > Sin Monitor > Intel Core i3 > 1 TB
genius mouse usb is probably part of: Computación > Periféricos de PC > Mouses > Ópticos > Genius > Con Cable
{% endhighlight %}
El resultado es interesante de analizar. En el ejemplo de la entrada "notebook bangho i3 4gb" el clasificador predijo que se trataba del conjunto de categorías: Computación > Notebooks y Accesorios > Notebooks > Bangho > Intel Core i3 > 15" a 15,9".

Vemos que todas las categorías son las correctas salvo la última, pero la entrada no decía nada que indique de cuantas pulgadas es la pantalla de la notebook. Lo que se entiende es que el clasificador asumió que lo más seguro es que si no se sabe exactamente de cuantas pulgadas es la pantalla, lo más probable es que la notebook sea de 15" ya que son las más comunes.
