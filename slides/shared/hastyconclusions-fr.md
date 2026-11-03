## Tirons des conclusions hâtives

* Le goulot d'étranglement (bottleneck) semble être `rng`

* *Et si* nous manquions d'entropie et ne parvenions pas à générer suffisamment de nombres aléatoires ?

* Nous devons faire un scale out du service `rng` sur plusieurs machines !

Note : il s'agit d'une fiction ! Nous disposons de suffisamment d'entropie. Mais nous avons besoin d'un prétexte pour réaliser ce scale out.

(En réalité, le code de `rng` utilise `/dev/urandom`, qui ne manque jamais d'entropie...
<br/>
... et qui est [tout aussi performant que `/dev/random`](http://www.slideshare.net/PacSecJP/filippo-plain-simple-reality-of-entropy).)