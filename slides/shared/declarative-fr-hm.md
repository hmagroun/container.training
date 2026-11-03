# Declarative vs imperative

- Notre orchestrateur de conteneurs met un accent tout particulier sur l'approche *déclarative*

- Déclaratif :

  *Je voudrais une tasse de thé.*

- Impératif :

  *Faites bouillir de l'eau. Versez-la dans une théière. Ajoutez les feuilles de thé. Laissez infuser un moment. Servez dans une tasse.*

--

- Le déclaratif semble plus simple au premier abord...

--

- ... À condition de savoir comment préparer le thé

---

## Declarative vs imperative

- Ce que serait réellement une approche déclarative :

  *Je veux une tasse de thé, obtenue en versant une infusion¹ de feuilles de thé dans une tasse.*

--

  *¹Une infusion s'obtient en laissant reposer l'objet quelques minutes dans de l'eau chaude².*

--

  *²Un liquide chaud s'obtient en le versant dans un récipient³ adapté et en le plaçant sur une plaque chauffante.*

--

  *³Ah, enfin des conteneurs ! Un sujet que nous maîtrisons. Mettons-nous au travail, vous voulez bien ?*

--

.footnote[Saviez-vous qu'il existe une [norme ISO](https://en.wikipedia.org/wiki/ISO_3103)
spécifiant la méthode d'infusion du thé ?]

---

## Declarative vs imperative

- Systèmes impératifs :

  - plus simples

  - si une tâche est interrompue, nous devons repartir de zéro

- Systèmes déclaratifs :

  - si une tâche est interrompue (ou si nous arrivons en cours de route),
    nous pouvons déterminer ce qui manque et réaliser uniquement ce qui est nécessaire

  - nous devons être capables d'*observer* le système

  - ... et calculer un "diff" entre *l'état actuel* et *l'état désiré*