# how-to-install-sql-server

# Configuración de SQL Server con Laravel Herd en Windows

Guía paso a paso para conectar un proyecto Laravel (usando Herd) con SQL Server.

---

## Requisitos previos

- [Laravel Herd](https://herd.laravel.com/) instalado
- PHP 8.4 activo en Herd
- Docker Desktop instalado y corriendo
- Un proyecto Laravel creado

---

## 1. Levantar SQL Server en Docker

```powershell
docker run -e "ACCEPT_EULA=Y" -e "SA_PASSWORD=TuPassword123" `
  -p 1433:1433 --name mi-sqlserver `
  -d mcr.microsoft.com/mssql/server:2022-latest
```

Verifica que el contenedor esté corriendo:

```powershell
docker ps
```

Debes ver `mi-sqlserver` con estado `Up` y el puerto `0.0.0.0:1433->1433/tcp`.

---

## 2. Crear la base de datos

```powershell
docker exec -it mi-sqlserver /opt/mssql-tools18/bin/sqlcmd `
  -S localhost -U sa -P TuPassword123 -Q "CREATE DATABASE laravel" -C
```

> También puedes crearla desde DBeaver u otro cliente SQL si lo prefieres.

---

## 3. Instalar el ODBC Driver de Microsoft

Descarga e instala el **ODBC Driver 18 for SQL Server** desde:

```
https://aka.ms/downloadmsodbcsql
```

Elige la versión **Windows x64** e instala con todas las opciones por defecto.

Verifica que quedó instalado:

```powershell
Get-OdbcDriver | Select-Object Name | findstr "ODBC Driver"
```

Debes ver `ODBC Driver 18 for SQL Server` en la lista.

---

## 4. Descargar los DLL del driver PHP

Ve a las siguientes páginas y descarga el zip de **PHP 8.4 Non Thread Safe (NTS) x64**:

- `sqlsrv`: https://pecl.php.net/package/sqlsrv/5.13.1/windows
- `pdo_sqlsrv`: https://pecl.php.net/package/pdo_sqlsrv/5.13.1/windows

Descomprime los zips. De cada uno necesitas solo el `.dll`:

```
php_sqlsrv.dll
php_pdo_sqlsrv.dll
```

> El archivo `.pdb` no es necesario, puedes ignorarlo.

---

## 5. Identificar el php.ini activo

Abre PowerShell y corre:

```powershell
php --ini
```

Anota la ruta que aparece en **Loaded Configuration File**. Puede ser algo como:

```
C:\Users\TU_USUARIO\OneDrive\Documentos\PrimerProyecto\php\php.ini
```

> ⚠️ Herd puede tener múltiples php.ini. Siempre edita el que aparece en `php --ini`, no el de la carpeta de Herd directamente.

---

## 6. Copiar los DLL a la carpeta ext de Herd

```powershell
Copy-Item "RUTA\php_sqlsrv.dll" "C:\Users\TU_USUARIO\.config\herd\bin\php84\ext\"
Copy-Item "RUTA\php_pdo_sqlsrv.dll" "C:\Users\TU_USUARIO\.config\herd\bin\php84\ext\"
```

Verifica que se copiaron:

```powershell
ls "C:\Users\TU_USUARIO\.config\herd\bin\php84\ext\" | findstr sqlsrv
```

---

## 7. Agregar las extensiones al php.ini

Abre el `php.ini` que encontraste en el paso 5:

```powershell
notepad "RUTA\php.ini"
```

Agrega al final del archivo las rutas completas a los DLL:

```ini
extension=C:\Users\TU_USUARIO\.config\herd\bin\php84\ext\php_sqlsrv.dll
extension=C:\Users\TU_USUARIO\.config\herd\bin\php84\ext\php_pdo_sqlsrv.dll
```

> ⚠️ Usa la ruta completa, no solo el nombre del archivo. De lo contrario PHP no los encuentra.

Guarda el archivo.

---

## 8. Verificar que los drivers cargaron

```powershell
php -m | findstr sql
```

Debes ver en la lista:

```
pdo_sqlsrv
sqlsrv
```

Si no aparecen, reinicia Herd desde la bandeja del sistema y vuelve a intentarlo.

---

## 9. Configurar el .env de Laravel

Abre el archivo `.env` en la raíz de tu proyecto y ajusta:

```env
DB_CONNECTION=sqlsrv
DB_HOST=localhost
DB_PORT=1433
DB_DATABASE=laravel
DB_USERNAME=sa
DB_PASSWORD=TuPassword123
```

---

## 10. Ajustar config/database.php

Asegúrate de que el bloque `sqlsrv` en `config/database.php` tenga estas opciones:

```php
'sqlsrv' => [
    'driver'                   => 'sqlsrv',
    'host'                     => env('DB_HOST', 'localhost'),
    'port'                     => env('DB_PORT', '1433'),
    'database'                 => env('DB_DATABASE', ''),
    'username'                 => env('DB_USERNAME', ''),
    'password'                 => env('DB_PASSWORD', ''),
    'charset'                  => 'utf8',
    'prefix'                   => '',
    'prefix_indexes'           => true,
    'encrypt'                  => 'yes',
    'trust_server_certificate' => true,
],
```

Limpia el caché de configuración:

```powershell
php artisan config:clear
```

---

## 11. Probar la conexión

```powershell
php artisan tinker
```

```php
DB::connection()->getPdo();
```

Si ves un objeto `PDO` con `DRIVER_NAME: "sqlsrv"` y tu base de datos en `CurrentDatabase`, ¡todo está listo!

---

## 12. Correr las migraciones

```powershell
php artisan migrate
```

---

## Errores comunes

| Error | Causa | Solución |
|---|---|---|
| `could not find driver` | ODBC Driver no instalado o DLL no cargando | Verifica pasos 3, 6 y 7 |
| `Login failed for user 'sa'` | Credenciales incorrectas | Revisa `DB_PASSWORD` en `.env` |
| `Connection refused` | Docker no está corriendo | `docker start mi-sqlserver` |
| DLL no carga con nombre corto | PHP no encuentra la carpeta ext | Usa ruta completa en `php.ini` (paso 7) |
| `php --ini` muestra un php.ini diferente al de Herd | PATH apunta a otro PHP | Edita siempre el que muestra `php --ini` |
