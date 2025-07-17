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

## **Paso 1.3: Creando tu Primer Clúster**

El clúster es el motor computacional que procesará tus datos. Sin él, los notebooks no pueden ejecutar código.

### **Configuración del Clúster:**

1. **Navega a la sección “Compute”:**
- Haz clic en el ícono **“Compute”** en el menú lateral
1. **Crear un nuevo clúster:**
- Haz clic en **“Create Compute”**
1. **Configuración recomendada:**
- **Cluster Name:** `cluster-proyecto-github`
- **Databricks Runtime Version:** Selecciona la versión **LTS** más reciente (no Beta)
- **Worker Type:** Mantener configuración por defecto
- **Driver Type:** Mantener configuración por defecto
- **Autoscaling:** Mantener habilitado
1. **Iniciar el clúster:**
- Haz clic en **“Create Cluster”**
- El proceso toma entre 3-5 minutos
- **Estado listo:** Círculo verde junto al nombre

### **Limitaciones Community Edition:**

- ✅ **Permitido:** 1 clúster activo
- ✅ **Permitido:** Hasta 2 nodos worker
- ❌ **Limitado:** Sin acceso a funcionalidades premium
- ❌ **Limitado:** Sin almacenamiento persistente integrado

### **Material de Apoyo:**

- **Documentación Oficial:** [Guía sobre Clusters en Databricks](https://docs.databricks.com/clusters/index.html)

-----

## **Paso 1.4: Configurando Azure Storage Account**

Dado que Databricks Community Edition no permite almacenamiento persistente integrado, configuraremos Azure Storage Account como nuestro sistema de almacenamiento externo.

### **0. Validación del Acceso a Azure**

#### **Verificación de Permisos (Paso a Paso):**

1. **Acceder al grupo de recursos:**
   
   ```
   URL: https://portal.azure.com/#@tu-tenant.onmicrosoft.com/resource/subscriptions/tu-subscription-id/resourceGroups/rg-proyecto-github/overview
   ```
1. **Navegar a Access Control (IAM):**
- Menú lateral → **“Access control (IAM)”**
- Pestañas superiores visibles
1. **Verificar permisos específicos:**
- Sección **“My access”** → Botón **“View my access”**
- Panel derecho: **“assignments - rg-proyecto-github”**
1. **Confirmar roles asignados:**

|Role                             |Description                                         |Scope                   |Status     |
|---------------------------------|----------------------------------------------------|------------------------|-----------|
|**Reader**                       |View all resources, but does not allow modifications|Subscription (inherited)|✅ Requerido|
|**Storage Account Contributor**  |Lets you manage storage accounts and access keys    |This resource           |✅ Requerido|
|**Storage Blob Data Contributor**|Allows for read, write and delete access to blobs   |This resource           |✅ Requerido|

#### **Obtención de Credenciales:**

1. **Localizar Storage Account:**
- En el grupo de recursos, buscar recurso tipo **“Storage account”**
- Hacer clic en el nombre del Storage Account
1. **Acceder a las claves:**
- Menú lateral → **“Security + networking”** → **“Access keys”**
- Copiar **Storage account name**
- Revelar y copiar **Key 1** o **Key 2**

### **1. Configurar Databricks Secrets**

#### **Crear Secret Scope:**

1. **Navegar a secrets:**
   
   ```
   URL: https://tu-workspace.cloud.databricks.com/#secrets/createScope
   ```
1. **Configuración:**
- **Scope Name:** `azure-storage-secrets`
- **Manage Principal:** Creator
- **Backend Type:** Databricks

#### **Agregar Secrets:**

1. **Secret 1 - Nombre del Storage:**
   
   ```
   URL: https://tu-workspace.cloud.databricks.com/#secrets/createSecret
   ```
- **Scope:** `azure-storage-secrets`
- **Key:** `storage-account-name`
- **Value:** [Nombre del Storage Account]
1. **Secret 2 - Clave de Acceso:**
   
   ```
   URL: https://tu-workspace.cloud.databricks.com/#secrets/createSecret
   ```
- **Scope:** `azure-storage-secrets`
- **Key:** `storage-account-key`
- **Value:** [Clave completa del Storage Account]

### **2. Configurar Conexión en Databricks**

#### **Código de Configuración (Notebook):**

```python
# Verificar configuración de secrets
print("🔍 Verificando configuración de secrets...")
try:
    scopes = dbutils.secrets.listScopes()
    target_scope = "azure-storage-secrets"
    
    if target_scope in [scope.name for scope in scopes]:
        secrets = dbutils.secrets.list(target_scope)
        secret_keys = [secret.key for secret in secrets]
        print(f"✅ Scope encontrado: {target_scope}")
        print(f"🔑 Secrets disponibles: {secret_keys}")
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
        print(f"✅ Conexión exitosa! Contenedores: {len(containers)}")
        
        return blob_service_client
        
    except Exception as e:
        print(f"❌ Error de conexión: {e}")
        return None

# Inicializar cliente
storage_client = get_storage_client()
```

### **3. Crear Estructura del Proyecto**

#### **Configurar Medallion Architecture:**

```python
# Configuración del proyecto
container_name = "alopez-proyecto-gh"
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
1. **Clúster Computacional**
- ✅ Clúster creado con configuración óptima
- ✅ Runtime LTS seleccionado
- ✅ Estado activo (círculo verde)
1. **Azure Storage Account**
- ✅ Permisos verificados en Portal Azure
- ✅ Credenciales obtenidas
- ✅ Databricks Secrets configurados
- ✅ Conexión establecida
1. **Estructura del Proyecto**
- ✅ Contenedor `alopez-proyecto-gh` creado
- ✅ Medallion Architecture implementada:
  - 🥉 **Bronze:** Datos en bruto
  - 🥈 **Silver:** Datos procesados
  - 🥇 **Gold:** Datos para análisis

### **🎯 Próximo Paso:**

**[➡️ Fase 2: Carga y Almacenamiento (Capa Bronze)](./fase-2-capa-bronze.md)**

### **📚 Material de Apoyo:**

- **Databricks:** [Documentación Oficial](https://docs.databricks.com/getting-started/index.html)
- **Azure Storage:** [Guía de Python SDK](https://docs.microsoft.com/en-us/azure/storage/blobs/storage-quickstart-blobs-python)
- **Medallion Architecture:** [Conceptos y mejores prácticas](https://databricks.com/glossary/medallion-architecture)

