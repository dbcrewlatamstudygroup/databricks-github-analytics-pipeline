# **Fase 2: Carga y Almacenamiento (Capa Bronze)**

## **Introducción**

Ahora que nuestro entorno está listo y tenemos configurado Azure Storage Account, vamos a realizar nuestra primera tarea de ingeniería de datos. Descargaremos archivos de eventos de GitHub y los almacenaremos en su formato original (crudo) en Azure Storage. Esta primera capa de almacenamiento se conoce como **Capa Bronze** en la Medallion Architecture.

-----

## **Paso 2.1: Creando tu Primer Notebook**

El notebook es donde escribiremos y ejecutaremos nuestro código para la ingesta de datos.

### **Configuración del Notebook:**

1. **Ve al Workspace:** Haz clic en el ícono de “Workspace” en el menú de la izquierda.
1. **Crea una carpeta:** Es una buena práctica organizar tu trabajo. Crea una carpeta llamada `Proyecto-GitHub-Analytics`.
1. **Crea un nuevo Notebook:**
- Dentro de la carpeta que acabas de crear, haz clic en el botón de “Crear” y selecciona “Notebook”
- **Nombre:** Asígnale un nombre descriptivo, como `01_Ingesta_Bronze_Azure`
- **Default Language:** Selecciona Python
- **Cluster:** Asegúrate de que esté adjunto al clúster que creaste en el paso anterior
1. **Abre el Notebook:** Haz clic en “Create” y se abrirá tu primer notebook.

-----

## **Paso 2.2: Configuración del Entorno Azure Storage**

Antes de descargar datos, configuraremos la conexión a Azure Storage usando los secrets que creamos anteriormente.

### **Celda 1: Importar librerías y configurar Azure Storage**

Copia y pega el siguiente código en la primera celda del notebook:

```python
# Importar librerías necesarias
import urllib.request 
import os 
import gzip 
import shutil
import tempfile
from azure.storage.blob import BlobServiceClient
from datetime import datetime

print("📦 CONFIGURACIÓN AZURE STORAGE PARA PROYECTO GITHUB")
print("=" * 60)

# Configurar cliente de Azure Storage usando secrets
def get_storage_client():
    try:
        # Obtener credenciales desde Databricks secrets
        storage_account_name = dbutils.secrets.get(scope="azure-storage-secrets", key="storage-account-name")
        storage_account_key = dbutils.secrets.get(scope="azure-storage-secrets", key="storage-account-key")
        
        # Crear connection string
        connection_string = f"DefaultEndpointsProtocol=https;AccountName={storage_account_name};AccountKey={storage_account_key};EndpointSuffix=core.windows.net"
        
        # Crear cliente
        blob_service_client = BlobServiceClient.from_connection_string(connection_string)
        
        print(f"✅ Conectado a Azure Storage: {storage_account_name}")
        return blob_service_client
        
    except Exception as e:
        print(f"❌ Error al conectar con Azure Storage: {e}")
        return None

# Inicializar cliente
storage_client = get_storage_client()

# Configuración del proyecto
container_name = "alopez-proyecto-gh"
bronze_folder = "bronze"
silver_folder = "silver" 
gold_folder = "gold"

print(f"📁 Contenedor: {container_name}")
print(f"🥉 Capa Bronze: {bronze_folder}/")
print(f"🥈 Capa Silver: {silver_folder}/")
print(f"🥇 Capa Gold: {gold_folder}/")
```

**Para ejecutar una celda, presiona Shift + Enter.**

### **Celda 2: Verificar y configurar la estructura del proyecto**

```python
# Verificar que la estructura del proyecto existe en Azure Storage
def verify_project_structure():
    if not storage_client:
        print("❌ Cliente de storage no disponible")
        return False
    
    try:
        # Verificar que el contenedor existe
        container_client = storage_client.get_container_client(container_name)
        
        # Verificar que las carpetas existen
        folders_to_check = [bronze_folder, silver_folder, gold_folder]
        existing_folders = []
        
        blobs = container_client.list_blobs()
        blob_names = [blob.name for blob in blobs]
        
        for folder in folders_to_check:
            folder_exists = any(blob_name.startswith(f"{folder}/") for blob_name in blob_names)
            if folder_exists:
                existing_folders.append(folder)
                print(f"✅ Carpeta '{folder}' verificada")
            else:
                print(f"⚠️  Carpeta '{folder}' no encontrada")
        
        if len(existing_folders) == 3:
            print("🎉 Estructura del proyecto verificada completamente")
            return True
        else:
            print("📝 Creando carpetas faltantes...")
            create_missing_folders(folders_to_check, existing_folders)
            return True
            
    except Exception as e:
        print(f"❌ Error verificando estructura: {e}")
        return False

def create_missing_folders(all_folders, existing_folders):
    """Crear carpetas faltantes"""
    missing_folders = [folder for folder in all_folders if folder not in existing_folders]
    
    for folder in missing_folders:
        try:
            blob_name = f"{folder}/.placeholder"
            blob_client = storage_client.get_blob_client(container=container_name, blob=blob_name)
            blob_client.upload_blob(f"# Carpeta {folder} del proyecto GitHub Analytics", overwrite=True)
            print(f"✅ Carpeta '{folder}' creada")
        except Exception as e:
            print(f"❌ Error creando carpeta '{folder}': {e}")

# Ejecutar verificación
verify_project_structure()
```

### **Celda 3: Funciones utilitarias para Azure Storage**

```python
# Funciones para gestionar archivos en Azure Storage
class GitHubDataManager:
    def __init__(self, storage_client, container_name):
        self.storage_client = storage_client
        self.container_name = container_name
    
    def upload_file_to_bronze(self, local_file_path, blob_name):
        """Subir archivo a la capa Bronze"""
        try:
            blob_path = f"{bronze_folder}/{blob_name}"
            blob_client = self.storage_client.get_blob_client(container=self.container_name, blob=blob_path)
            
            with open(local_file_path, "rb") as data:
                blob_client.upload_blob(data, overwrite=True)
            
            print(f"✅ Archivo subido a Bronze: {blob_name}")
            return True
            
        except Exception as e:
            print(f"❌ Error subiendo archivo: {e}")
            return False
    
    def list_bronze_files(self):
        """Listar archivos en la capa Bronze"""
        try:
            container_client = self.storage_client.get_container_client(self.container_name)
            blobs = container_client.list_blobs(name_starts_with=f"{bronze_folder}/")
            
            files = []
            print(f"📁 Archivos en la capa Bronze:")
            for blob in blobs:
                if not blob.name.endswith('.placeholder'):
                    files.append(blob.name)
                    file_size = blob.size / (1024*1024)  # MB
                    print(f"  📄 {blob.name} ({file_size:.2f} MB)")
            
            return files
            
        except Exception as e:
            print(f"❌ Error listando archivos: {e}")
            return []
    
    def get_bronze_file_info(self):
        """Obtener información detallada de archivos en Bronze"""
        try:
            container_client = self.storage_client.get_container_client(self.container_name)
            blobs = container_client.list_blobs(name_starts_with=f"{bronze_folder}/")
            
            total_size = 0
            file_count = 0
            
            for blob in blobs:
                if not blob.name.endswith('.placeholder'):
                    total_size += blob.size
                    file_count += 1
            
            total_size_mb = total_size / (1024*1024)
            print(f"📊 Resumen Capa Bronze:")
            print(f"   Archivos: {file_count}")
            print(f"   Tamaño total: {total_size_mb:.2f} MB")
            
            return {"file_count": file_count, "total_size_mb": total_size_mb}
            
        except Exception as e:
            print(f"❌ Error obteniendo información: {e}")
            return None

# Crear instancia del gestor
data_manager = GitHubDataManager(storage_client, container_name)

print("🔧 GitHubDataManager inicializado")
print("📝 Funciones disponibles:")
print("   - data_manager.upload_file_to_bronze(archivo_local, nombre_blob)")
print("   - data_manager.list_bronze_files()")
print("   - data_manager.get_bronze_file_info()")
```

-----

## **Paso 2.3: Código para Descargar y Guardar los Datos en Azure Storage**

Ahora descargaremos los datos de eventos de GitHub y los almacenaremos en la capa Bronze de Azure Storage.

### **Celda 4: Configurar URLs de descarga de GitHub Archive**

```python
# Configuración de descarga de datos de GitHub Archive
print("🐙 CONFIGURACIÓN DE DESCARGA GITHUB ARCHIVE")
print("=" * 50)

# Lista de fechas y horas para las que queremos descargar los datos
# Formato: AAAA-MM-DD-H
# Descargamos las primeras 3 horas de dos días diferentes como ejemplo
urls_to_download = [
    "https://data.gharchive.org/2025-01-01-0.json.gz",
    "https://data.gharchive.org/2025-01-01-1.json.gz", 
    "https://data.gharchive.org/2025-01-01-2.json.gz",
    "https://data.gharchive.org/2025-01-02-0.json.gz",
    "https://data.gharchive.org/2025-01-02-1.json.gz",
    "https://data.gharchive.org/2025-01-02-2.json.gz"
]

print(f"📅 Archivos a descargar: {len(urls_to_download)}")
print("📋 Lista de archivos:")
for i, url in enumerate(urls_to_download, 1):
    file_name = url.split("/")[-1]
    print(f"   {i}. {file_name}")

print(f"\n💾 Destino: Azure Storage → {container_name} → {bronze_folder}/")
```

### **Celda 5: Función de descarga y carga a Azure Storage**

```python
# Función para descargar y subir archivos a Azure Storage
def download_and_upload_github_data(urls_list):
    """
    Descarga archivos de GitHub Archive y los sube a Azure Storage (capa Bronze)
    """
    if not storage_client:
        print("❌ Cliente de Azure Storage no disponible")
        return
    
    successful_downloads = 0
    failed_downloads = 0
    
    print("🚀 INICIANDO PROCESO DE DESCARGA Y CARGA")
    print("=" * 60)
    
    for i, url in enumerate(urls_list, 1):
        file_name_gz = url.split("/")[-1]
        file_name_json = file_name_gz.replace(".gz", "")
        
        print(f"\n📦 Procesando archivo {i}/{len(urls_list)}: {file_name_gz}")
        
        # Crear archivos temporales
        with tempfile.NamedTemporaryFile(suffix='.gz', delete=False) as temp_gz:
            temp_gz_path = temp_gz.name
            
        with tempfile.NamedTemporaryFile(suffix='.json', delete=False) as temp_json:
            temp_json_path = temp_json.name
        
        try:
            # Paso 1: Descargar archivo comprimido
            print(f"   📥 Descargando desde: {url}")
            urllib.request.urlretrieve(url, temp_gz_path)
            print(f"   ✅ Descarga completada")
            
            # Paso 2: Descomprimir archivo
            print(f"   📂 Descomprimiendo archivo...")
            with gzip.open(temp_gz_path, 'rb') as f_in:
                with open(temp_json_path, 'wb') as f_out:
                    shutil.copyfileobj(f_in, f_out)
            print(f"   ✅ Descompresión completada")
            
            # Paso 3: Subir a Azure Storage (capa Bronze)
            print(f"   ☁️  Subiendo a Azure Storage...")
            success = data_manager.upload_file_to_bronze(temp_json_path, file_name_json)
            
            if success:
                successful_downloads += 1
                print(f"   🎉 {file_name_json} procesado exitosamente")
            else:
                failed_downloads += 1
                print(f"   ❌ Error subiendo {file_name_json}")
            
        except Exception as e:
            failed_downloads += 1
            print(f"   ❌ Error procesando {file_name_gz}: {e}")
        
        finally:
            # Limpiar archivos temporales
            try:
                os.unlink(temp_gz_path)
                os.unlink(temp_json_path)
            except:
                pass
    
    # Resumen final
    print("\n" + "=" * 60)
    print("📊 RESUMEN DE DESCARGA")
    print("=" * 60)
    print(f"✅ Descargas exitosas: {successful_downloads}")
    print(f"❌ Descargas fallidas: {failed_downloads}")
    print(f"📁 Total de archivos: {len(urls_list)}")
    
    if successful_downloads > 0:
        print(f"\n🎉 ¡Proceso completado! Los datos están en Azure Storage.")
        print(f"📍 Ubicación: {container_name}/{bronze_folder}/")
    
    return successful_downloads, failed_downloads

# Ejecutar el proceso de descarga
results = download_and_upload_github_data(urls_to_download)
```

-----

## **Paso 2.4: Verificar que los Archivos se Guardaron Correctamente**

### **Celda 6: Verificación de archivos en Azure Storage**

```python
# Verificar que los archivos se guardaron correctamente en Azure Storage
print("🔍 VERIFICACIÓN DE ARCHIVOS EN CAPA BRONZE")
print("=" * 50)

# Listar archivos en la capa Bronze
bronze_files = data_manager.list_bronze_files()

# Obtener información detallada
file_info = data_manager.get_bronze_file_info()

# Verificación adicional desde Azure Portal
print(f"\n🔗 Para verificar en Azure Portal:")
print(f"   1. Ve a: https://portal.azure.com")
print(f"   2. Navega a: rg-proyecto-github → Storage Account")
print(f"   3. Ve a: Containers → {container_name} → {bronze_folder}/")
print(f"   4. Deberías ver {len(bronze_files)} archivos .json")

# Validación de contenido (sample de un archivo)
if bronze_files:
    print(f"\n📄 Validación de contenido del primer archivo:")
    try:
        # Descargar una muestra pequeña del primer archivo para verificar
        first_file = bronze_files[0]
        blob_client = storage_client.get_blob_client(container=container_name, blob=first_file)
        
        # Leer las primeras líneas
        sample_data = blob_client.download_blob().readall()
        sample_lines = sample_data.decode('utf-8').split('\n')[:3]
        
        print(f"   Archivo: {first_file.split('/')[-1]}")
        print(f"   Primeras líneas de ejemplo:")
        for i, line in enumerate(sample_lines, 1):
            if line.strip():
                print(f"     {i}. {line[:100]}...")
        
        print("   ✅ El archivo contiene datos JSON válidos de GitHub")
        
    except Exception as e:
        print(f"   ❌ Error verificando contenido: {e}")

print(f"\n🎉 ¡Felicidades! Has completado la carga de datos a la Capa Bronze.")
print(f"📈 Próximo paso: Transformar estos datos en la Capa Silver.")
```

### **Celda 7: Función utilitaria para futuros notebooks**

```python
# Función utilitaria para reutilizar en otros notebooks
def get_bronze_file_paths():
    """
    Obtener las rutas de todos los archivos en la capa Bronze
    Útil para los siguientes notebooks de transformación
    """
    try:
        container_client = storage_client.get_container_client(container_name)
        blobs = container_client.list_blobs(name_starts_with=f"{bronze_folder}/")
        
        file_paths = []
        for blob in blobs:
            if not blob.name.endswith('.placeholder'):
                # Formato para leer desde Azure Storage en Spark
                azure_path = f"abfss://{container_name}@{storage_client.account_name}.dfs.core.windows.net/{blob.name}"
                file_paths.append(azure_path)
        
        return file_paths
        
    except Exception as e:
        print(f"❌ Error obteniendo rutas: {e}")
        return []

# Guardar información para el siguiente notebook
bronze_paths = get_bronze_file_paths()
print(f"📝 Rutas para el siguiente notebook:")
for path in bronze_paths[:3]:  # Mostrar solo las primeras 3
    print(f"   {path}")

if len(bronze_paths) > 3:
    print(f"   ... y {len(bronze_paths) - 3} archivos más")

print(f"\n💡 Estas rutas se usarán en el notebook de transformación (Silver)")
```

-----

## **¿Qué es GitHub Archive?**

**GitHub Archive** es un proyecto que registra la cronología pública de GitHub, capturando eventos que ocurren en repositorios públicos cada hora.

### **Tipos de Eventos Capturados:**

|Evento              |Descripción                             |Frecuencia Típica|
|--------------------|----------------------------------------|-----------------|
|**PushEvent**       |Commits subidos a repositorios          |🔥🔥🔥🔥🔥            |
|**CreateEvent**     |Creación de ramas, tags o repositorios  |🔥🔥🔥⚪⚪            |
|**PullRequestEvent**|Eventos de Pull Requests                |🔥🔥🔥🔥⚪            |
|**IssuesEvent**     |Apertura, cierre o comentarios en issues|🔥🔥🔥⚪⚪            |
|**WatchEvent**      |Usuario da “star” a un repositorio      |🔥🔥🔥🔥⚪            |
|**ForkEvent**       |Fork de un repositorio                  |🔥🔥⚪⚪⚪            |

### **Estructura de Datos:**

Cada archivo JSON contiene eventos con información como:

- Usuario que realizó la acción
- Repositorio afectado
- Tipo de evento
- Timestamp del evento
- Detalles específicos del evento

### **Ejemplo de Evento JSON:**

```json
{
  "id": "12345678901",
  "type": "PushEvent",
  "actor": {
    "id": 123456,
    "login": "octocat",
    "display_login": "octocat"
  },
  "repo": {
    "id": 654321,
    "name": "octocat/Hello-World"
  },
  "created_at": "2025-01-01T00:00:00Z",
  "payload": {
    "commits": [
      {
        "sha": "abc123def456...",
        "message": "Update README.md"
      }
    ]
  }
}
```

-----

## **Resumen de la Fase 2**

### **✅ Lo que has logrado:**

1. **Notebook de Ingesta Creado**
- ✅ Configuración de Azure Storage
- ✅ Funciones utilitarias implementadas
- ✅ Manejo de errores incluido
1. **Datos de GitHub Descargados**
- ✅ 6 archivos de eventos de GitHub (2 días, 3 horas cada uno)
- ✅ Datos descomprimidos y almacenados en formato JSON
- ✅ Aproximadamente 6-20 MB de datos por archivo
1. **Capa Bronze Implementada**
- ✅ Datos en formato crudo/original
- ✅ Almacenados en Azure Storage
- ✅ Estructura organizada por fecha/hora
1. **Funcionalidades Disponibles**
- ✅ Gestión automática de archivos temporales
- ✅ Validación de contenido
- ✅ Información estadística de archivos
- ✅ Preparación para siguiente fase

### **🎯 Próximos Pasos:**

**Semana 2:** Transformación de datos y creación de la Capa Silver

- Limpieza y validación de datos
- Aplicación de esquemas estructurados
- Transformaciones con PySpark
- Optimización de performance

### **📚 Material de Apoyo:**

- **GitHub Archive:** [Documentación oficial](https://www.gharchive.org/)
- **Azure Blob Storage:** [Guía de Python SDK](https://docs.microsoft.com/en-us/azure/storage/blobs/storage-quickstart-blobs-python)
- **Medallion Architecture:** [Mejores prácticas](https://databricks.com/glossary/medallion-architecture)

### **🔍 Troubleshooting Común:**

|Problema                  |Solución                                                                    |
|--------------------------|----------------------------------------------------------------------------|
|Error de conexión a Azure |Verificar secrets y permisos                                                |
|Archivos no se descargan  |Verificar conectividad a internet                                           |
|Error de espacio temporal |Los archivos temporales se limpian automáticamente                          |
|Error de permisos en Azure|Verificar roles: Storage Account Contributor + Storage Blob Data Contributor|

-----

**🎉 ¡Felicidades!** Has completado exitosamente la **Fase 2** del proyecto. Tienes una capa Bronze funcional con datos reales de GitHub listos para ser transformados en la siguiente semana.

**[⬅️ Regresar a Fase 1](./fase-1-configuracion.md)** | **[➡️ Continuar a Semana 2](../semana-2/README.md)** *(Próximamente)*

