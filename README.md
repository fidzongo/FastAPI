# Gestionnaire de QCM avec FastAPI
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

# Authentification des utilisateurs
Une authentification basique, à base de nom d'utilisateur et de mot de passe est utilisée.
La chaîne de caractères contenant Basic username:password est passée dans l'en-tête Authorization.

# Code de l'API
Vous pouvez utiliser également le fichier main.py
```python
# Import des librairies
import pandas as pd
import json
import uvicorn
from fastapi import FastAPI, Security, Request, Depends, HTTPException, status
from pydantic import BaseModel
from typing import Optional, List
from fastapi.security.api_key import APIKeyHeader
from starlette.status import HTTP_401_UNAUTHORIZED


# Definition de la classe QCM
class QCM(BaseModel):
    use_type: str
    questions_number: Optional[int] = 5
    subject: List[str]


# Definition de la classe User
class User(BaseModel):
    username: str
    password: str


# Definition de la classe Uses
class Suject(BaseModel):
    use: str


# Definition de la classe Question
class Question(BaseModel):
    question: str	
    subject: str	
    use: str	
    correct: str	
    responseA: str	
    responseB: str	
    responseC: str	
    responseD: Optional[str] = None	
    remark: Optional[str] = None


# token pour surcharger le header Authorization
token_key = APIKeyHeader(name="Authorization")


# Declaration de la classe Token
class Token(BaseModel):
    token: str


# Declaration d'un objet fastapi
app = FastAPI(
    title="Evaluation Module FastAPI",
    description="Questionaire",
    version="1.0.0",
    openapi_tags=[
        {
            'name': 'PUBLIC',
            'description': "Point de terminaison permettant de vérifier que l'API est bien fonctionnelle"
        },
        {
            'name': 'QCM',
            'description': 'Visualisation: des uses cases, des sujets et génération des qcm'
        },
        {
            'name': 'ADMIN',
            'description': 'Administration: ajout des questions...'
        }
    ]    
    )


# dataframe / table des utilisateurs
users = {
    "alice":"wonderland",
    "bob":"builder",
    "clementine":"mandarine"
    }


# dataframe / table des administrateur
admins = {
    "admin":"4dm1N"
    }

# Permet de verifier si l'api est fonctionnelle
status = {'status': ['operationnel', 'maintenance']} 
df_status = pd.DataFrame.from_dict(status)


# Lecture du fichier csv
df = pd.read_csv('questions.csv', sep=';', header=0)
# suppression des nan
df.dropna(subset=['use', 'subject'], inplace=True)


def get_current_token(auth_key: str = Security(token_key)):
    '''
    Permet de recuperer la chaine Basic 'username:password' a passer au header Authorization
    '''
    return auth_key

 
def auth(request: Request):
    '''
    Recupere le header Authorization et retourne l'utilisateur et son mot de passe
    '''
    authorization_header = request.headers.get('Authorization')
    
    if authorization_header is None:
        return "Not permited"
    try:
        scheme, credentials = authorization_header.split(' ')
        username,password = credentials.split(':')
        return {"Authorization": authorization_header, "username": username, "password": password}
    except:
        return "Veuillez verifier la syntaxe de l'authentification Basic username:password"


def parse_csv(df):
    '''
    Permet de parser les donnees a afficher
    '''
    res = df.to_json(orient="records")
    parsed = json.loads(res)
    return parsed


@app.get('/status', name="Permet de tester si l'api est fonctionnelle", tags=['PUBLIC'])
async def get_status():
    '''
    Permet de tester si l'api est fonctionnelle.
    Cet endpoint est public et n'a pas besoin d'une authentification.
    '''
    if df_status['status'][0] == "operationnel":
        return {"L'API est fonctionnel"}
        #return {"status": parse_csv(df_status['status'])}
    else:
        return {"L'API n'est pas fonctionnel"}


@app.put('/users', tags=['PUBLIC'])
async def put_users(user: User):
    '''
    Permet la création de compte utilisateurs.
    Il n'est necessaire d'être authentifier pour cet endpoint
    Le mot string n'est pas authorisé pour le nom d'utilisateur
    Le mot de passe ne peut pas être vide
    '''
    if user.username in users:
        #return {"UserExist":"Ce utilisateur existe déjà. Veuillez choisir un autre utilisateur"}
        raise HTTPException(
            status_code=400,
            detail='Ce utilisateur existe déjà. Veuillez choisir un autre utilisateur')
    else:
        if not user.password.strip():
            #return {"Le mot de passe ne peut pas être null"}
            raise HTTPException(
                status_code=401,
                detail='Le mot de passe ne peut pas être null')
        elif user.username =="string":
            #return {"Veuillez saisir un utilisateur"}
            raise HTTPException(
                status_code=402,
                detail='Veuillez saisir un utilisateur valide')
        else:
            users.update({user.username:user.password})
            #return {"l'utilisateur "+user.username+" a été crée avec succès"}
            raise HTTPException(
                status_code=200,
                detail="l'utilisateur "+user.username+" a été crée avec succès")

@app.get('/use', name='Types de QCM disponibles', tags=['QCM'])
async def get_use(auth: str = Depends(auth), current_token: Token = Depends(get_current_token)):
    '''
    Permet de visualiser les cas d'utilisations qui existent dans la base
    Il faut être authentifier pour tester cet endpoint
    '''
    try:
        username = auth['username'] 
        password = auth['password']
    except:
        #return "Veuillez verifier la syntaxe de l'authentification : Basic username:password"
        raise HTTPException(
            status_code=403,
            detail="Veuillez verifier la syntaxe de l'authentification : Basic username:password")

    if username in users:
        if users[username] == password:
            #return {"use": parse_csv(df['use'].drop_duplicates().str.strip())}
            return {"use": parse_csv(df['use'].drop_duplicates().str.strip())}
        else:
            raise HTTPException(
            status_code=401,
            detail="Incorrect username or password")
            #return {"Incorrect username or password"}
    elif username in admins:
        if admins[username] == password:
            return {"use": parse_csv(df['use'].drop_duplicates().str.strip())}
        else:
            raise HTTPException(
            status_code=401,
            detail="Incorrect username or password")
            #return {"Incorrect username or password"}
    else:
        raise HTTPException(
        status_code=401,
        detail="Incorrect username or password")


@app.get('/subject', name='Liste des subjets disponibles pour chaque use case', tags=['QCM'])
async def get_subject(use, auth: str = Depends(auth), current_token: Token = Depends(get_current_token)):
    '''
    Permet de lister les sujets disponibles en base
    Le champs use est obligatoire
    Il faut être authentifier pour tester cet endpoint
    '''
    if not use:
        return {"Le champs use est obligatoire"}
    else:	
        try:
            username = auth['username'] 
            password = auth['password']
        except:
            #return "Veuillez verifier la syntaxe de l'authentification : Basic username:password"
            raise HTTPException(
                status_code=403,
                detail="Veuillez verifier la syntaxe de l'authentification : Basic username:password")

        if username in users:
            if users[username] == password:
                #df1 = df[['subject','use']] [(df['use'] == use)]
                #df1.drop_duplicates(subset='subject', keep="first")
                #df1.replace('^\s+', '', regex=True, inplace=True)
                #df1.replace('\s+$', '', regex=True, inplace=True)
                #return {'subject': parse_csv(df1['subject'])}
                df1 = df[['subject','use']] [(df['use'] == use)]
                return {'subject': parse_csv(df1['subject'].drop_duplicates().str.strip())}
            else:
                raise HTTPException(
				status_code=401,
				detail="Incorrect username or password")
				#return {"Incorrect username or password"}
        elif username in admins:
            if admins[username] == password:
                return {"use": parse_csv(df['use'].drop_duplicates().str.strip())}
            else:
                raise HTTPException(
				status_code=401,
				detail="Incorrect username or password")
				#return {"Incorrect username or password"}
        else:
            raise HTTPException(
			status_code=401,
			detail="Incorrect username or password")


@app.post('/qcm', name='Generateur de QCM', tags=['QCM'])
async def post_qcm(qcm: QCM, auth: str = Depends(auth), current_token: Token = Depends(get_current_token)):
    '''
    Permet de generer aleatoirement QCM avec un ensemble de questions
    Il faut être authentifier pour tester cet endpoint
    '''
    data = df[['use','subject','question','responseA','responseB','responseC','responseD']] [(df['use'].str.strip() == qcm.use_type) & (df['subject'].str.strip().isin(qcm.subject))]
    try:
        username = auth['username'] 
        password = auth['password']
    except:
        #return "Veuillez verifier la syntaxe de l'authentification : Basic username:password"
        raise HTTPException(
            status_code=403,
            detail="Veuillez verifier la syntaxe de l'authentification : Basic username:password")

    if username in users:
        if users[username] == password:
            try:
                return parse_csv(data.sample(n=qcm.questions_number))
				#ValueError: "Cannot take a larger sample than population when 'replace=False'"
            except ValueError as error:
				#print(error)
				#ValueError: a must be greater than 0 unless no samples are taken
                return parse_csv(data.sample(n=len(data)))
				#return parse_csv(data.sample(n=qcm.questions_number))
        else:
            raise HTTPException(
            status_code=401,
            detail="Incorrect username or password")
            #return {"Incorrect username or password"}
    elif username in admins:
        if admins[username] == password:
            return {"use": parse_csv(df['use'].drop_duplicates().str.strip())}
        else:
            raise HTTPException(
            status_code=401,
            detail="Incorrect username or password")
            #return {"Incorrect username or password"}
    else:
        raise HTTPException(
        status_code=401,
        detail="Incorrect username or password")


@app.put('/questions', name='Ajout de question', tags=['ADMIN'])
async def put_questions(question: Question, auth: str = Depends(auth), current_token: Token = Depends(get_current_token)):
    '''
    Permet aux administrateurs d'ajoute de nouvelles questions
    '''
    try:
        username = auth['username'] 
        password = '4dm1N'
    except:
        #return "Veuillez verifier la syntaxe de l'authentification : Basic username:password"
        raise HTTPException(
            status_code=403,
            detail="Veuillez verifier la syntaxe de l'authentification : Basic username:password")

    if username in admins:
        if admins[username] == password:
            new_question = {"question":question.question, "subject":question.subject, "use":question.use, "correct":question.correct, "responseA":question.responseA, "responseB":question.responseB, "responseC":question.responseC, "responseD":question.responseD, "remark":question.remark}
            global df
            df = df.append(new_question, ignore_index=True)
            return {"question":new_question}
        else:
            raise HTTPException(
            status_code=401,
            detail="Incorrect username or password")
            #return {"Incorrect username or password"}
    else:
        raise HTTPException(
        status_code=401,
        detail="Vous n'êtes pas authoriser a ajouter des questions")
#df


#uvicorn.run(f"{__name__}:app", host="0.0.0.0", port=9000, reload=False)
```

# Tests / Utilisation de l'API

## Ajout d’un utilisateur
curl -X 'PUT' \
  'http://localhost:8000/users' \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -d '{
  "username": "XXXX",
  "password": "XXXX"
}'

## Affichage des uses disponibles
curl -X 'GET' \
  'http://localhost:8000/use' \
  -H 'accept: application/json' \
  -H 'Authorization: Basic XXXX:XXXX'

## Affichage des subjects disponibles
curl -X 'POST' \
  'http://localhost:8000/qcm' \
  -H 'accept: application/json' \
  -H 'Authorization: Basic XXXX:XXXX' \
  -H 'Content-Type: application/json' \
  -d '{
  "use_type": "Test de validation",
  "questions_number": 5,
  "subject": [
    "Classification"
  ]
}'

## Génération d’un QCM de 5 questions (on peut le relancer plusieurs fois pour voir que les questions affichent aléatoirement)
curl -X 'POST' \
  'http://localhost:8000/qcm' \
  -H 'accept: application/json' \
  -H 'Authorization: Basic XXXX:XXXX' \
  -H 'Content-Type: application/json' \
  -d '{
  "use_type": "Test de positionnement",
  "questions_number": 5,
  "subject": [
    "BDD", "Docker"

  ]
}'

## Ajout d’une question
curl -X 'PUT' \
  'http://localhost:8000/questions' \
  -H 'accept: application/json' \
  -H 'Authorization: Basic XXXX:XXXX' \
  -H 'Content-Type: application/json' \
  -d '{
  "question": "FastAPI Permet il de",
  "subject": "FastAPI",
  "use": "FastAPI",
  "correct": "Mettre en place des API rapide",
  "responseA": "De gagner de l'\''argent",
  "responseB": "Marcher vite",
  "responseC": "Voyager",
  "responseD": "Permet une documentation automatique",
  "remark": "Ajout question"
}'

