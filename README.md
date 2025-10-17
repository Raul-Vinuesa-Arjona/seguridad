# seguridad
Seguridad por usuario y rol. Permisos de acceso a medidas y dimensiones
Se deja un pbix con datos dummy para dar un ejemplo.

# 🧩 Documentación. Modelo de seguridad por permisos en Power BI
Este enfoque implementa seguridad a nivel de usuario y permiso sin depender de roles RLS, lo que permite mantener la interactividad total de los informes y la compatibilidad con parámetros de campo, slicers y jerarquías.
🧱 1. Tabla_Control_Usuarios
Es el núcleo del sistema.

No se relaciona con el resto del modelo y sirve únicamente para definir los permisos de visibilidad que tiene cada usuario.
Contiene las columnas:
• Email → identifica al usuario (utilizado con USERPRINCIPALNAME()).
• Rol → agrupa usuarios con el mismo conjunto de permisos (opcional, solo informativo o para slicers).
• Permiso → nombre del permiso (por ejemplo, Ver_SALDO_TOTAL, Ver_dim_STAGE, etc.).
• ValorPermiso → valor 1 si tiene acceso, 0 si no.
Cada usuario tiene una fila por permiso, lo que hace la tabla totalmente escalable y fácil de mantener:

para añadir un nuevo permiso, solo hay que agregar las filas correspondientes con el nuevo nombre de permiso y el valor de acceso.
📋 2. Tabla Permisos
Se genera automáticamente con los valores únicos de [Permiso] desde Tabla_Control_Usuarios, mediante:

Permisos = DISTINCT(SELECTCOLUMNS(Tabla_Control_Usuarios, "Permiso", Tabla_Control_Usuarios[Permiso]))

Esto elimina la necesidad de mantener manualmente la lista de permisos y garantiza que siempre esté sincronizada con los permisos reales definidos.
⚙️ 3. Medida [PermisoUsuario]
Evalúa el permiso solicitado en el contexto del usuario actual:

PermisoUsuario =
VAR _usuario = USERPRINCIPALNAME()
VAR _permisoSolicitado = SELECTEDVALUE('Permisos'[Permiso])
VAR _valorPermiso =
    CALCULATE(
        MAX('Tabla_Control_Usuarios'[ValorPermiso]),
        'Tabla_Control_Usuarios'[Email] = _usuario,
        'Tabla_Control_Usuarios'[Permiso] = _permisoSolicitado
    )
RETURN
    _valorPermiso

Lógica:
• Detecta el usuario conectado (USERPRINCIPALNAME()).
• Toma el permiso activo en el contexto (desde la tabla Permisos).
• Busca en la tabla de control si ese usuario tiene ese permiso y devuelve 1 (permitido) o 0/BLANK (restringido).
📊 4. Uso en medidas y slicers
Para controlar la visibilidad de una medida o campo, se aplica este patrón:

Medida Visible =
IF(
    CALCULATE([PermisoUsuario], 'Permisos'[Permiso] = "Ver_SALDO_TOTAL") = 1,
    [SALDO TOTAL_Real],
    BLANK()
)

Así, las medidas solo se muestran si el usuario tiene el permiso correspondiente, sin afectar filtros, relaciones o comportamiento visual.
🧠 5. Ventajas del enfoque
✅ No rompe relaciones ni jerarquías.

✅ Totalmente compatible con parámetros de campo.

✅ Escalable y fácil de mantener (solo se añaden filas en la tabla de control).

✅ Permite gestionar permisos granulares (por medida, por dimensión, por grupo, etc.).

✅ No requiere publicar o administrar roles RLS en el servicio.
En resumen, este modelo implementa una capa de seguridad dinámica y autorreferencial basada en permisos definidos en tabla, con medidas DAX que adaptan el contenido visible según el usuario autenticado, manteniendo la máxima flexibilidad y rendimiento dentro del modelo semántico.
