# FastAPI
Mise en place d'un gestionneur de QCM avec FastAPI.

FastAPI :
- Un outil très utile pour créer des APIs. 
- Possède une documentation exhaustive et facile à prendre en main
- Utilisation de OpenAPI (anciennement Swagger) comme documentation et permet de facilement mettre en place des tests.

# Contenu

# Contexte
Il s'agit ici de mettre en place une application qui vas permettre d'intérroger une base de données pour retourner une série de questions sous forme de QCM. L'application permet également d'ajouter ou de modifier des questions. La base de données est représentée par le fichier csv questions.csv

Sur l'application l'utilisateur choisit un type de test (use) ainsi qu'une ou plusieurs catégories (subject). De plus, l'application peut produire des QCMs de 5, 10 ou 20 questions. L'API doit donc être en mesure de retourner ce nombre de questions. Comme l'application doit pouvoir générer de nombreux QCMs, les questions doivent être retournées dans un ordre aléatoire: ainsi, une requête avec les mêmes paramètres pourra retourner des questions différentes.

L'application doit permettre l'utilisation d'un compte pour pouvoir générer les QCM. Si le compte utilisateur n'existe pas il doit permettre à l'utilisation de créer son compte.

# Installation de FastAPI
Pour installer FastAPI
```sh
pip3 install fastapi uvicorn
ou 
pip install -r requirements.txt
```
