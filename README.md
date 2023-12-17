# EjercicioGCP

Documento Json en el Cloud Storage  
![image](https://github.com/cristianolaya1/EjercicioGCP/assets/145338030/7d9bade2-b20e-4b3f-9ac0-8d4de33ab6df)

Visualización del documento Json  
![image](https://github.com/cristianolaya1/EjercicioGCP/assets/145338030/70a273e2-120e-468a-97bf-c750e0c429f0)

Formato de guardado del documento en la base de datos  
![image](https://github.com/cristianolaya1/EjercicioGCP/assets/145338030/f0b3df98-beec-4cc1-8261-2d29447ef733)


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
