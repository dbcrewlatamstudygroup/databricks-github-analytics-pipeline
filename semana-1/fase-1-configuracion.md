# **Fase 1: Configuración del Entorno en Databricks**

## **Introducción**

Antes de poder construir cualquier pipeline de datos, necesitamos preparar nuestro espacio de trabajo. Esto implica registrarnos en Databricks Community Edition, crear un clúster computacional, y configurar Azure Storage Account como nuestro sistema de almacenamiento externo. Esta configuración nos permitirá implementar la **Medallion Architecture** (Bronze-Silver-Gold) para nuestro proyecto de análisis de datos de GitHub.

-----

## **Paso 1.1: Registrarse en Databricks Community Edition**

La Community Edition es una versión gratuita de Databricks, ideal para aprender y desarrollar proyectos personales.

### **Proceso de Registro:**

1. **Ve a la página de registro:**
- Abre tu navegador y dirígete a: [Databricks Community Edition](https://community.cloud.databricks.com/login.html)
1. **Completa el formulario:**
- Rellena tus datos personales
- Asegúrate de usar un correo electrónico válido
- Crea una contraseña segura
1. **Selecciona la opción “Community Edition”:**
- Cuando se te presente la opción de elegir un plan
- Selecciona **“Get started with Community Edition”**
1. **Verifica tu correo electrónico:**
- Recibirás un correo para confirmar tu cuenta
- Sigue las instrucciones para activar tu workspace

### **Material de Apoyo:**

- **Documentación Oficial:** [Guía de inicio de Databricks](https://docs.databricks.com/getting-started/community-edition.html)

-----

## **Paso 1.2: Explorando la Interfaz de Databricks**

Una vez dentro de tu workspace, familiarízate con la interfaz principal.

### **Componentes Principales:**

|Sección      |Descripción                            |Uso en el Proyecto                |
|-------------|---------------------------------------|----------------------------------|
|**Workspace**|Organiza carpetas, notebooks y archivos|Crear estructura del proyecto     |
|**Compute**  |Gestiona clústeres computacionales     |Crear y administrar el clúster    |
|**Data**     |Explora bases de datos y tablas        |Visualizar datos transformados    |
|**Jobs**     |Automatiza ejecución de notebooks      |Para etapas avanzadas del proyecto|

### **Navegación Básica:**

- **Menú lateral izquierdo:** Acceso rápido a todas las secciones
- **Área principal:** Espacio de trabajo principal
- **Barra superior:** Configuraciones de usuario y workspace

-----

## **Paso 1.3: Verificando tu Entorno Computacional**

Databricks Community Edition viene con un **Serverless Starter Warehouse** preconfigurado que actuará como nuestro motor computacional para procesar datos.

### **Verificación del Serverless Warehouse:**

1. **Navega a la sección "Compute":**
   - Haz clic en el ícono **"Compute"** en el menú lateral

2. **Verificar el Serverless Warehouse:**
   - Deberías ver un warehouse llamado **"Starter Warehouse"** o similar
   - **Estado:** Debe aparecer como "Running" o con un círculo verde
   - **Tipo:** Serverless (Sin necesidad de configuración manual)

3. **Si el warehouse no está activo:**
   - Haz clic en el warehouse para seleccionarlo
   - Clic en **"Start"** si está detenido
   - El proceso toma 1-2 minutos en iniciarse

4. **Verificación completa:**
   - **Estado activo:** Círculo verde junto al nombre
   - **Tipo:** Serverless Starter Warehouse
   - **Listo para usar:** Sin configuración adicional necesaria

### **Limitaciones Community Edition:**
- ✅ **Permitido:** Serverless Starter Warehouse (Ya creado)
- ❌ **Limitado:** Sin acceso a funcionalidades premium
- ❌ **Limitado:** Sin almacenamiento persistente integrado

### **Material de Apoyo:**
- **Documentación Oficial:** [Databricks Serverless Compute](https://docs.databricks.com/compute/serverless.html)

-----

## **Paso 1.4: Configurando Azure Storage Account**

Dado que Databricks Community Edition no permite almacenamiento persistente integrado, configuraremos Azure Storage Account como nuestro sistema de almacenamiento externo.

### **0. Validación del Acceso a Azure**

#### **Información del Proyecto:**
- **Grupo de Recursos:** `rg-proyecto-github`
- **Storage Account:** `stgaccproyectogh`

#### **Verificación de Permisos (Paso a Paso):**

1. **Acceder al grupo de recursos:**
   - Ve a: [Azure Portal](https://portal.azure.com)
   - En la barra de búsqueda superior, escribe: `rg-proyecto-github`
   - Haz clic en el grupo de recursos **"rg-proyecto-github"** cuando aparezca en los resultados
   - Esto te llevará al dashboard del grupo de recursos

2. **Navegar a Access Control (IAM):**
   - Menú lateral → **"Access control (IAM)"**
   - Pestañas superiores visibles

3. **Verificar permisos específicos:**
   - Sección **"My access"** → Botón **"View my access"**
   - Panel derecho: **"assignments - rg-proyecto-github"**

4. **Confirmar roles asignados:**

| Role | Description | Scope | Status |
|------|-------------|-------|--------|
| **Reader** | View all resources, but does not allow modifications | Subscription (inherited) | ✅ Requerido |
| **Storage Account Contributor** | Lets you manage storage accounts and access keys | This resource | ✅ Requerido |
| **Storage Blob Data Contributor** | Allows for read, write and delete access to blobs | This resource | ✅ Requerido |

#### **Obtención de Credenciales:**

1. **Localizar Storage Account:**
   - En el grupo de recursos, buscar el recurso **"stgaccproyectogh"** (tipo: Storage account)
   - Hacer clic en el nombre del Storage Account: **stgaccproyectogh**

2. **Acceder a las claves:**
   - Menú lateral → **"Security + networking"** → **"Access keys"**
   - Copiar **Storage account name:** `stgaccproyectogh`
   - Revelar y copiar **Key 1** o **Key 2**

### **1. Configurar Databricks Secrets**

#### **💡 Enfoque Híbrido: UI + Python SDK**

**Ventajas de este método:**
- ✅ **UI para Scope:** Visual y fácil de entender
- ✅ **SDK para Secrets:** Programático y reproducible  
- ✅ **Sin limitaciones:** No depende de URLs específicas
- ✅ **Más control:** Manejo directo de errores
- ✅ **Documentable:** Código que se puede versionar

Utilizaremos la **interfaz web** para crear el scope (más visual) y **Python SDK** para agregar secrets (más programático y reproducible).

#### **Crear Secret Scope (Desde la UI):**

1. **Navegar a secrets en Databricks:**
   - En tu workspace de Databricks, agrega `#secrets/createScope` al final de la URL
   - URL: `https://tu-workspace.cloud.databricks.com/#secrets/createScope`

2. **Configurar el Scope:**
   - **Scope Name:** `azure-storage-secrets`
   - **Manage Principal:** Creator (por defecto)
   - Haz clic en **"Create"**

#### **Agregar Secrets (Usando Python SDK):**

Una vez creado el scope, ejecuta este código en una celda de tu notebook:

```python
# Instalar Databricks SDK si no está disponible
%pip install databricks-sdk

# Importar y configurar el cliente
from databricks.sdk import WorkspaceClient

# Inicializar cliente de Databricks
w = WorkspaceClient()

# Agregar el nombre del Storage Account
w.secrets.put_secret(
    scope="azure-storage-secrets",
    key="storage-account-name", 
    string_value="stgaccproyectogh"
)

# Agregar la clave de acceso del Storage Account
# IMPORTANTE: Reemplaza "TU_STORAGE_ACCOUNT_KEY" con la clave real obtenida del Portal Azure
w.secrets.put_secret(
    scope="azure-storage-secrets",
    key="storage-account-key",
    string_value="TU_STORAGE_ACCOUNT_KEY"  # ← Reemplazar con la clave real
)

print("✅ Secrets configurados exitosamente:")
print("   - storage-account-name: stgaccproyectogh")
print("   - storage-account-key: [OCULTA]")
```

#### **Verificar Secrets Configurados:**

```python
# Verificar que los secrets se crearon correctamente
try:
    # Listar scopes
    scopes = w.secrets.list_scopes()
    target_scope = "azure-storage-secrets"
    
    if any(scope.name == target_scope for scope in scopes):
        print(f"✅ Scope '{target_scope}' encontrado")
        
        # Listar secrets en el scope
        secrets = w.secrets.list_secrets(scope=target_scope)
        secret_keys = [secret.key for secret in secrets]
        print(f"🔑 Secrets disponibles: {secret_keys}")
        
        # Verificar que tenemos los secrets necesarios
        required_keys = ["storage-account-name", "storage-account-key"]
        if all(key in secret_keys for key in required_keys):
            print("🎉 Todos los secrets requeridos están configurados")
        else:
            missing = [key for key in required_keys if key not in secret_keys]
            print(f"❌ Faltan estos secrets: {missing}")
    else:
        print(f"❌ Scope '{target_scope}' no encontrado")
        
except Exception as e:
    print(f"❌ Error verificando secrets: {e}")
```

### **2. Configurar Conexión en Databricks**

#### **Código de Configuración (Notebook):**

```python
# Verificar configuración de secrets usando dbutils (método tradicional)
print("🔍 Verificando configuración de secrets...")
try:
    scopes = dbutils.secrets.listScopes()
    target_scope = "azure-storage-secrets"
    
    if target_scope in [scope.name for scope in scopes]:
        secrets = dbutils.secrets.list(target_scope)
        secret_keys = [secret.key for secret in secrets]
        print(f"✅ Scope encontrado: {target_scope}")
        print(f"🔑 Secrets disponibles: {secret_keys}")
        
        # Verificar que tenemos los secrets necesarios
        required_keys = ["storage-account-name", "storage-account-key"]
        if all(key in secret_keys for key in required_keys):
            print("🎉 Todos los secrets requeridos están configurados")
        else:
            missing = [key for key in required_keys if key not in secret_keys]
            print(f"❌ Faltan estos secrets: {missing}")
    else:
        print(f"❌ Scope '{target_scope}' no encontrado")
        
except Exception as e:
    print(f"❌ Error: {e}")
```

```python
# Configurar cliente de Azure Storage
%pip install azure-storage-blob

from azure.storage.blob import BlobServiceClient

def get_storage_client():
    try:
        # Obtener credenciales desde secrets
        storage_account_name = dbutils.secrets.get(scope="azure-storage-secrets", key="storage-account-name")
        storage_account_key = dbutils.secrets.get(scope="azure-storage-secrets", key="storage-account-key")
        
        # Crear connection string
        connection_string = f"DefaultEndpointsProtocol=https;AccountName={storage_account_name};AccountKey={storage_account_key};EndpointSuffix=core.windows.net"
        
        # Crear cliente
        blob_service_client = BlobServiceClient.from_connection_string(connection_string)
        
        # Test de conexión
        containers = list(blob_service_client.list_containers())
        print(f"✅ Conectado a Azure Storage: {storage_account_name}")
        print(f"📦 Contenedores encontrados: {len(containers)}")
        
        return blob_service_client
        
    except Exception as e:
        print(f"❌ Error de conexión: {e}")
        return None

# Inicializar cliente
storage_client = get_storage_client()
```

### **3. Crear Estructura del Proyecto**

En este paso vas a crear un contenedor con la siguiente sintaxis:
> primeraLetraNombrePrimerApellido-proyecto-gh -->> Ejemplo alopez-proyecto-gh


Validar que no exista otro contenedor ya creado con este nombre, si es asi utiliza la siguiente sintaxis:
> primeraLetraNombrePrimerApellidoPrimeraLetraSegundoApellido-proyecto-gh -->> Ejemplo alopezm-proyecto-gh

#### **Configurar Medallion Architecture:**

```python
# Configuración del proyecto
container_name = "nombre-contenedor"
folders = ["bronze", "silver", "gold"]

def setup_project_structure():
    if not storage_client:
        print("❌ Cliente no disponible")
        return False
    
    try:
        # Crear contenedor
        try:
            container_client = storage_client.create_container(container_name)
            print(f"✅ Contenedor '{container_name}' creado")
        except Exception as e:
            if "ContainerAlreadyExists" in str(e):
                print(f"ℹ️  Contenedor '{container_name}' ya existe")
        
        # Crear carpetas (Bronze, Silver, Gold)
        for folder in folders:
            blob_name = f"{folder}/.placeholder"
            blob_client = storage_client.get_blob_client(container=container_name, blob=blob_name)
            blob_client.upload_blob(f"# Carpeta {folder} - Medallion Architecture", overwrite=True)
            print(f"✅ Carpeta '{folder}' creada")
        
        return True
        
    except Exception as e:
        print(f"❌ Error: {e}")
        return False

# Ejecutar configuración
setup_project_structure()
```

### **4. Verificar Configuración Completa**

#### **Checklist de Validación:**

```python
# Verificación final de la configuración
def verify_complete_setup():
    print("🔍 VERIFICACIÓN COMPLETA DEL ENTORNO")
    print("=" * 50)
    
    checks = []
    
    # Check 1: Databricks Secrets
    try:
        dbutils.secrets.get("azure-storage-secrets", "storage-account-name")
        checks.append("✅ Databricks Secrets configurados")
    except:
        checks.append("❌ Databricks Secrets faltantes")
    
    # Check 2: Azure Storage Connection
    if storage_client:
        checks.append("✅ Conexión a Azure Storage")
    else:
        checks.append("❌ Sin conexión a Azure Storage")
    
    # Check 3: Estructura del proyecto
    try:
        container_client = storage_client.get_container_client(container_name)
        blobs = list(container_client.list_blobs())
        if len(blobs) >= 3:
            checks.append("✅ Estructura Medallion creada")
        else:
            checks.append("❌ Estructura Medallion incompleta")
    except:
        checks.append("❌ Error verificando estructura")
    
    # Mostrar resultados
    for check in checks:
        print(f"   {check}")
    
    all_good = all("✅" in check for check in checks)
    
    if all_good:
        print("\n🎉 ¡Configuración completa y exitosa!")
        print("📍 Listo para la Fase 2: Carga de datos")
    else:
        print("\n⚠️  Hay elementos por configurar")
        print("📍 Revisa los elementos marcados con ❌")

# Ejecutar verificación
verify_complete_setup()
```

-----

## **Resumen de la Fase 1**

### **✅ Elementos Configurados:**

1. **Databricks Community Edition**
   - ✅ Cuenta creada y verificada
   - ✅ Workspace activo
   - ✅ Interfaz explorada

2. **Serverless Warehouse**
   - ✅ Starter Warehouse verificado y activo
   - ✅ Estado "Running" confirmado
   - ✅ Listo para procesar datos

3. **Azure Storage Account**
   - ✅ Permisos verificados en Portal Azure
   - ✅ Storage Account `stgaccproyectogh` configurado  
   - ✅ Databricks Secrets configurados (UI + SDK)
   - ✅ Conexión establecida y probada

4. **Estructura del Proyecto**
   - ✅ Contenedor `nombre-contenedor` creado
   - ✅ Medallion Architecture implementada:
     - 🥉 **Bronze:** Datos en bruto
     - 🥈 **Silver:** Datos procesados
     - 🥇 **Gold:** Datos para análisis

### **🎯 Próximo Paso:**

**[➡️ Fase 2: Carga y Almacenamiento (Capa Bronze)](./fase-2-capa-bronze.md)**

### **📚 Material de Apoyo:**

- **Databricks:** [Documentación Oficial](https://docs.databricks.com/getting-started/index.html)
- **Databricks SDK:** [Python SDK Documentation](https://databricks-sdk-py.readthedocs.io/)
- **Azure Storage:** [Guía de Python SDK](https://docs.microsoft.com/en-us/azure/storage/blobs/storage-quickstart-blobs-python)
- **Medallion Architecture:** [Conceptos y mejores prácticas](https://databricks.com/glossary/medallion-architecture)