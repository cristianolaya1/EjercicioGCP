# EjercicioGCP

Cloudbuild.yaml para crear la cloud function, contenedor, subir la imagen

```
steps:
- name: 'gcr.io/cloud-builders/gcloud'
  args:
  - functions
  - deploy
  - read_json_from_gcs
  - --runtime=python39
  - --region=europe-west1
  - --trigger-bucket=gs://jercicio_gcp
  - --allow-unauthenticated
  - --source=./mi_app/cloud_function

#Contenedor
- name: 'gcr.io/cloud-builders/docker'
  args: ['build', '-t', 'europe-west1-docker.pkg.dev/proyecto-thebridge/repo/webapp-run', './mi_app/cloud_run/']

#Subida
- name: 'gcr.io/cloud-builders/docker'
  args: ['push', 'europe-west1-docker.pkg.dev/proyecto-thebridge/repo/webapp-run']

#Despliega la imagen en una cloud run
- name: 'gcr.io/cloud-builders/gcloud'
  entrypoint: gcloud
  args:
  - run
  - deploy
  - webapp-run
  - --image=europe-west1-docker.pkg.dev/proyecto-thebridge/repo/webapp-run
  - --region=europe-west1
  - --platform=managed
  - --allow-unauthenticated
  - --port=8080
images:
- 'europe-west1-docker.pkg.dev/proyecto-thebridge/repo/webapp-run'

#Despliega la imagen en una cloud run
- name: 'gcr.io/cloud-builders/gcloud'
  entrypoint: gcloud
  args:
  - run
  - deploy
  - webapp-run
  - --image=europe-west1-docker.pkg.dev/proyecto-thebridge/repo/webapp-run
  - --region=europe-west1
  - --platform=managed
  - --allow-unauthenticated
  - --port=8080
images:
- 'europe-west1-docker.pkg.dev/mi-proyecto-thebridge/mi-repositorio/webapp-run'
```

Codigo de la cloud function
```
import json
from google.cloud import storage, firestore

def mover(event, context):
    storage_client = storage.Client()
    bucket_name = event['bucket']
    file_name = event['name']

    print(f"Cubo {bucket_name} y archivo {file_name} encontrados")

    if file_name.endswith('.json'):

        # Descargar el contenido del archivo JSON desde Cloud Storage
        bucket = storage_client.bucket(bucket_name)
        blob = bucket.blob(file_name)
        json_content = json.loads(blob.download_as_text())

        print("Json descargado")

        # Insertar el contenido en Firestore
        firestore_client = firestore.Client()
        collection_name = 'usuarios'  # Reemplaza con el nombre de tu colección
        document_name = file_name.split('.')[0]  # Utilizar el nombre del archivo como ID del documento

        print("Json insertado en la base de datos")

        # Asumiendo que el contenido del archivo JSON es un diccionario, puedes ajustar esto según tu estructura
        firestore_client.collection(collection_name).document(document_name).set(json_content)

        print(f"Archivo {file_name} insertado correctamente en la base de datos.")
    else:
        print(f"No se ha encontrado nigun Json: {file_name} en el cubo {bucket_name}")

```

Codigo de la aplicacion web
```python
import json
import boto3
import dash
from dash import dcc, html, dash_table
import random
import datetime

# Creamos una aplicación Dash
app = dash.Dash(__name__)

# Configuramos la conexión a la tabla de DynamoDB
dynamodb = boto3.resource('dynamodb', region_name='eu-west-3') #colocar la region que corresponde
tabla_usuarios = dynamodb.Table('formulario') #colocar nopmbre que le hayas puesto a la tabla

# Función para obtener los datos de la tabla de DynamoDB
def obtener_datos_dynamodb():
    response = tabla_usuarios.scan()
    items = response['Items']
    return items

# Definimos el estilo CSS para la página
external_stylesheets = ['https://codepen.io/chriddyp/pen/bWLwgP.css']
app = dash.Dash(__name__, external_stylesheets=external_stylesheets)

# Definimos el diseño general de la aplicación
app.layout = html.Div(style={'backgroundColor': 'white', 'padding': '20px', 'text-align': 'center'}, children=[
    html.Div([
        html.H1('Cloud App', style={'color': 'black'}),
    ], style={'display': 'flex', 'align-items': 'center', 'justify-content': 'center', 'margin-bottom': '20px'}),

    # Menú de navegación
    html.Div([
        dcc.Link(html.Button('Formulario de Usuarios', id='btn-formulario', n_clicks=0,
                             style={'background-color': 'black', 'color': 'white', 'border': 'none', 'margin': '10px', 'box-shadow': '2px 2px 5px 0px #000000'}),
                 href='/formulario'),
        dcc.Link(html.Button('Tabla de Usuarios', id='btn-tabla-usuarios', n_clicks=0,
                             style={'background-color': 'black', 'color': 'white', 'border': 'none', 'margin': '10px', 'box-shadow': '2px 2px 5px 0px #000000'}),
                 href='/tabla_usuarios'),
    ], style={'display': 'flex', 'justify-content': 'center'}),

    # Aquí se mostrará el contenido
    dcc.Location(id='url', refresh=False),
    html.Div(id='page-content'),
])

# Callback para cargar el contenido de las páginas
@app.callback(
    dash.dependencies.Output('page-content', 'children'),
    [dash.Input('url', 'pathname')]
)
def display_page(pathname):
    if pathname == '/formulario':
        # Si el usuario navega al formulario, muestra el contenido del formulario
        return html.Div([
            html.H1('Formulario de Usuarios', style={'color': 'black'}),
            dcc.Input(id='nombre', type='text', placeholder='Nombre', value='', style={'margin-bottom': '10px'}),
            dcc.Input(id='email', type='email', placeholder='Email', value='', style={'margin-bottom': '10px'}),
            html.Button('Enviar', id='submit-button', n_clicks=0,
                        style={'background-color': 'black', 'color': 'white', 'border': 'none', 'box-shadow': '2px 2px 5px 0px #000000'}),
            html.Div(id='output-container-button', children='', style={'margin-top': '10px', 'color': 'black'})
        ])
    elif pathname == '/tabla_usuarios':
        # Si el usuario navega a la tabla de usuarios, muestra el contenido de la tabla
        data = obtener_datos_dynamodb()
        return html.Div([
            html.H1('Usuarios registrados', style={'color': 'black'}),
            dash_table.DataTable(
                columns=[{'name': key, 'id': key} for key in data[0].keys()],
                data=data,
                style_table={'overflowX': 'auto', 'border': '1px solid black', 'backgroundColor': 'white'},
                style_header={'backgroundColor': 'black', 'color': 'white', 'fontWeight': 'bold'},
                style_cell={'textAlign': 'left', 'border': '1px solid black'},
                style_data={'border': '1px solid black'},
                style_as_list_view=True
            )
        ])
# Ruta para manejar la subida de datos del formulario
@app.callback(
    dash.dependencies.Output('output-container-button', 'children'),
    [dash.Input('submit-button', 'n_clicks'),
     dash.State('nombre', 'value'),
     dash.State('email', 'value')]
)
def submit_form(n_clicks, nombre, email):
    # Obtenemos los datos del formulario
    usuario = {
        'ID': random.randint(100000, 999999),
        'Nombre': nombre,
        'Correo electrónico': email,
        'Fecha de registro': datetime.date.today().strftime('%Y-%m-%d')
    }
    tabla_usuarios.put_item(Item=usuario)
    return f'Se ha enviado el formulario: {nombre}, {email}'

if __name__ == '__main__':
    app.run_server(host='0.0.0.0', port=80, debug=True)
```

Documento docker
```
FROM python:3.7-alpine
COPY app /app
WORKDIR /app
RUN pip install -r requirements.txt
EXPOSE 8080

CMD ["python","app.py"]
```
