
-- =====================================================================
-- BEL Comercial R1 · Respuesta a "Dudas - Revisión DDL entidad relación"
-- (Domingo Camargo / Alfredo Pinzón)
-- Motor: Aurora MySQL (compatible MySQL 8)
-- Orden de ejecución: primero ddl_relaciones_belc_aurora.sql, luego este.
-- Convención: la columna de estado se llama siempre "Estado".
-- =====================================================================


-- #####################################################################
-- PUNTO 1 · Instancias de aprobación transaccional
-- La transacción en espera vive en este esquema. La configuración del
-- flujo (Flujo_aprobacion, _regla, _nivel, _aprobador) ya existe; faltaba
-- dónde guardar cada transacción concreta y sus decisiones.
-- Si negocio confirma que las transacciones no entran en el R1, estas
-- tablas quedan diseñadas y se crean en el release que las incluya.
-- #####################################################################

CREATE TABLE Transaccion_aprobacion (
  id_transaccion_aprobacion BIGINT        NOT NULL AUTO_INCREMENT,
  id_grupo_economico        INT           NOT NULL,
  id_empresa_asociada       INT           NOT NULL,
  id_flujo                  INT           NOT NULL,
  Version_flujo             INT           NOT NULL,       -- versión del flujo vigente al crearla
  id_regla                  INT           NOT NULL,       -- regla elegida por tipo + monto + moneda
  id_nivel_actual           INT           NULL,           -- NULL cuando ya no espera aprobación
  id_tipo_transaccion       INT           NOT NULL,
  id_producto_vinculacion   INT           NOT NULL,       -- cuenta origen
  Cuenta_destino            VARCHAR(40)   NULL,
  Monto                     DECIMAL(18,2) NOT NULL,
  id_moneda                 INT           NOT NULL,
  Datos_transaccion         JSON          NOT NULL,       -- detalle completo para ejecutar en el core
  Estado                    VARCHAR(15)   NOT NULL DEFAULT 'PENDIENTE',
  Solicitado_por            INT           NOT NULL,       -- id_usuario_grupo_economico del creador
  Fecha_solicitud           DATETIME(3)   NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
  Fecha_expiracion          DATETIME(3)   NOT NULL,       -- según Parametro_configuracion
  Fecha_resolucion          DATETIME(3)   NULL,
  Referencia_core           VARCHAR(50)   NULL,
  Motivo_fallo              VARCHAR(400)  NULL,
  id_correlacion            VARCHAR(64)   NOT NULL,
  Version                   INT           NOT NULL DEFAULT 1,
  CONSTRAINT pk_transaccion_aprobacion PRIMARY KEY (id_transaccion_aprobacion),
  CONSTRAINT ck_ta_estado CHECK (Estado IN ('PENDIENTE','EN_APROBACION','APROBADA','RECHAZADA',
                                            'EXPIRADA','CANCELADA','EJECUTADA','FALLIDA')),
  CONSTRAINT ck_ta_monto CHECK (Monto > 0),
  CONSTRAINT ck_ta_ejecutada CHECK (Estado <> 'EJECUTADA' OR Referencia_core IS NOT NULL),
  CONSTRAINT ck_ta_fallida CHECK (Estado <> 'FALLIDA' OR Motivo_fallo IS NOT NULL),
  CONSTRAINT ck_ta_nivel CHECK (Estado NOT IN ('PENDIENTE','EN_APROBACION') OR id_nivel_actual IS NOT NULL),
  CONSTRAINT fk_ta_empresa_grupo FOREIGN KEY (id_empresa_asociada, id_grupo_economico)
    REFERENCES Empresas_asociadas (id_empresa_asociada, id_grupo_economico),
  CONSTRAINT fk_ta_flujo FOREIGN KEY (id_flujo) REFERENCES Flujo_aprobacion (id_flujo),
  CONSTRAINT fk_ta_regla FOREIGN KEY (id_regla) REFERENCES Flujo_aprobacion_regla (id_regla),
  CONSTRAINT fk_ta_nivel FOREIGN KEY (id_nivel_actual) REFERENCES Flujo_aprobacion_nivel (id_nivel),
  CONSTRAINT fk_ta_tipo FOREIGN KEY (id_tipo_transaccion) REFERENCES Tipo_transaccion (id_tipo_transaccion),
  CONSTRAINT fk_ta_producto FOREIGN KEY (id_producto_vinculacion) REFERENCES Producto_vinculacion (id_producto_vinculado),
  CONSTRAINT fk_ta_moneda FOREIGN KEY (id_moneda) REFERENCES moneda (id_moneda),
  CONSTRAINT fk_ta_solicitante FOREIGN KEY (Solicitado_por) REFERENCES Usuario_grupo_economico (id_usuario_grupo_economico),
  INDEX ix_ta_estado_expiracion (Estado, Fecha_expiracion),   -- proceso que vence pendientes
  INDEX ix_ta_grupo_estado (id_grupo_economico, Estado)        -- bandeja de aprobaciones
) ENGINE = InnoDB DEFAULT CHARSET = utf8mb4;

CREATE TABLE Transaccion_aprobacion_decision (
  id_decision                BIGINT       NOT NULL AUTO_INCREMENT,
  id_transaccion_aprobacion  BIGINT       NOT NULL,
  id_nivel                   INT          NOT NULL,
  id_usuario_grupo_economico INT          NOT NULL,       -- el aprobador en esa relación
  Decision                   VARCHAR(10)  NOT NULL,
  Comentario                 VARCHAR(400) NULL,
  Fecha_decision_utc         DATETIME(3)  NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
  CONSTRAINT pk_ta_decision PRIMARY KEY (id_decision),
  CONSTRAINT uq_ta_decision_usuario UNIQUE (id_transaccion_aprobacion, id_usuario_grupo_economico), -- nadie aprueba dos veces
  CONSTRAINT ck_ta_decision CHECK (Decision IN ('APROBADA','RECHAZADA')),
  CONSTRAINT fk_tad_transaccion FOREIGN KEY (id_transaccion_aprobacion)
    REFERENCES Transaccion_aprobacion (id_transaccion_aprobacion),
  CONSTRAINT fk_tad_nivel FOREIGN KEY (id_nivel) REFERENCES Flujo_aprobacion_nivel (id_nivel),
  CONSTRAINT fk_tad_usuario FOREIGN KEY (id_usuario_grupo_economico)
    REFERENCES Usuario_grupo_economico (id_usuario_grupo_economico)
) ENGINE = InnoDB DEFAULT CHARSET = utf8mb4;

-- Bitácora: id_aprobacion no dice a qué aprobación pertenece. Se separa en dos.
ALTER TABLE BITACORA_EVENTO
  RENAME COLUMN id_aprobacion TO id_aprobacion_administrativa;
ALTER TABLE BITACORA_EVENTO
  ADD COLUMN id_transaccion_aprobacion BIGINT NULL AFTER id_aprobacion_administrativa,
  ADD CONSTRAINT ck_bitacora_una_aprobacion
    CHECK (id_aprobacion_administrativa IS NULL OR id_transaccion_aprobacion IS NULL);

-- Regla de configuración: la versión publicada de un flujo no se edita.
-- Un cambio de reglas, niveles o aprobadores crea una nueva Version del
-- flujo; así Version_flujo apunta siempre a reglas que no cambiaron.


-- #####################################################################
-- PUNTO 2 · Columna de versión en las tablas destino
-- Aprobacion_administrativa guarda Version_registro; cada tabla destino
-- necesita su Version para comparar. Regla: se versiona el registro
-- padre, y un cambio en su detalle sube la versión del padre.
--   Grupo_producto      <- Grupo_producto_detalle
--   role                <- Rol_permiso, Rol_grupo_producto
--   Flujo_aprobacion    <- _regla, _nivel, _aprobador, _producto (ya tiene Version)
-- Ya la tienen por el script de relaciones: Usuario_grupo_economico,
-- Grupo_economico_administrador, afiliacion, belc_persona_canal,
-- Contacto_notificacion.
-- #####################################################################

ALTER TABLE Grupo_economico      ADD COLUMN Version INT NOT NULL DEFAULT 1;
ALTER TABLE Empresas_asociadas   ADD COLUMN Version INT NOT NULL DEFAULT 1;
ALTER TABLE Producto_vinculacion ADD COLUMN Version INT NOT NULL DEFAULT 1;
ALTER TABLE Grupo_producto       ADD COLUMN Version INT NOT NULL DEFAULT 1;
ALTER TABLE role                 ADD COLUMN Version INT NOT NULL DEFAULT 1;
ALTER TABLE perfil               ADD COLUMN Version INT NOT NULL DEFAULT 1;

-- Uso al aplicar una aprobación (ejemplo con role):
-- UPDATE role
--    SET Nombre_role = :nombre, Version = Version + 1
--  WHERE id_role = :id_registro AND Version = :version_registro;
-- Si ROW_COUNT() = 0 -> otro cambio llegó antes: la aprobación pasa a FALLIDA.


-- #####################################################################
-- PUNTO 3 · Valores de Estado con CHECK
-- Antes de cada CHECK se normalizan los datos existentes (mayúsculas).
-- afiliacion, Usuario_grupo_economico, Grupo_economico_administrador,
-- belc_persona_canal y Contacto_notificacion ya los traen en su CREATE.
-- #####################################################################

-- Estandarizar el nombre de la columna de estado a "Estado" en las tablas
-- que hoy la tienen en minúscula (según el diagrama). MySQL 8 permite
-- renombrar solo cambiando mayúsculas.
ALTER TABLE Tipo_cambio_administrativo RENAME COLUMN estado TO Estado;
ALTER TABLE Tipo_producto              RENAME COLUMN estado TO Estado;
ALTER TABLE moneda                     RENAME COLUMN estado TO Estado;

-- Aprobacion_administrativa
UPDATE Aprobacion_administrativa SET Estado = UPPER(Estado), Accion = UPPER(Accion);
ALTER TABLE Aprobacion_administrativa
  ADD CONSTRAINT ck_aa_estado CHECK (Estado IN ('PENDIENTE','APROBADA','RECHAZADA','EXPIRADA',
                                                'CANCELADA','APLICADA','FALLIDA')),
  ADD CONSTRAINT ck_aa_accion CHECK (Accion IN ('CREAR','MODIFICAR','ELIMINAR')),
  ADD CONSTRAINT ck_aa_requeridas CHECK (Aprobaciones_requeridas >= 1),
  ADD CONSTRAINT ck_aa_fallida CHECK (Estado <> 'FALLIDA' OR Motivo_fallo IS NOT NULL);

-- Aprobacion_administrativa_decision: la persona por id_persona y sin doble aprobación
-- (pensado para el DDL en diseño; si la tabla ya tiene datos, primero migra
--  Identity_id a id_persona con un UPDATE ... JOIN belc_persona_canal)
ALTER TABLE Aprobacion_administrativa_decision
  ADD COLUMN id_persona INT NOT NULL AFTER id_aprobacion,
  DROP COLUMN Identity_id,
  ADD CONSTRAINT fk_aad_persona FOREIGN KEY (id_persona) REFERENCES belc_persona_canal (id_persona),
  ADD CONSTRAINT uq_aad_persona UNIQUE (id_aprobacion, id_persona),
  ADD CONSTRAINT ck_aad_decision CHECK (Decision IN ('APROBADA','RECHAZADA'));

-- Producto_vinculacion (Estado_core no lleva CHECK: refleja los valores del core)
UPDATE Producto_vinculacion SET Estado_Vinculacion = UPPER(Estado_Vinculacion);
ALTER TABLE Producto_vinculacion
  ADD CONSTRAINT ck_pv_estado CHECK (Estado_Vinculacion IN ('PENDIENTE','ACTIVO','SUSPENDIDO','DESVINCULADO')),
  ADD CONSTRAINT ck_pv_desvinculado CHECK (Estado_Vinculacion <> 'DESVINCULADO' OR Fecha_desvinculacion IS NOT NULL);

-- Catálogos y tablas de configuración: ACTIVO / INACTIVO
UPDATE Tipo_cambio_administrativo SET Estado = UPPER(Estado);
ALTER TABLE Tipo_cambio_administrativo ADD CONSTRAINT ck_tca_estado CHECK (Estado IN ('ACTIVO','INACTIVO'));

UPDATE Tipo_visualizacion SET Estado = UPPER(Estado);
ALTER TABLE Tipo_visualizacion ADD CONSTRAINT ck_tv_estado CHECK (Estado IN ('ACTIVO','INACTIVO'));

UPDATE Tipo_producto SET Estado = UPPER(Estado);
ALTER TABLE Tipo_producto ADD CONSTRAINT ck_tp_estado CHECK (Estado IN ('ACTIVO','INACTIVO'));

UPDATE permiso SET Estado = UPPER(Estado);
ALTER TABLE permiso ADD CONSTRAINT ck_permiso_estado CHECK (Estado IN ('ACTIVO','INACTIVO'));

UPDATE perfil SET Estado = UPPER(Estado);
ALTER TABLE perfil ADD CONSTRAINT ck_perfil_estado CHECK (Estado IN ('ACTIVO','INACTIVO'));

UPDATE role SET Estado = UPPER(Estado);
ALTER TABLE role ADD CONSTRAINT ck_role_estado CHECK (Estado IN ('ACTIVO','INACTIVO'));

UPDATE moneda SET Estado = UPPER(Estado);
ALTER TABLE moneda ADD CONSTRAINT ck_moneda_estado CHECK (Estado IN ('ACTIVO','INACTIVO'));

UPDATE Grupo_producto SET Estado = UPPER(Estado);
ALTER TABLE Grupo_producto ADD CONSTRAINT ck_gp_estado CHECK (Estado IN ('ACTIVO','INACTIVO'));

UPDATE Flujo_aprobacion SET Estado = UPPER(Estado);
ALTER TABLE Flujo_aprobacion ADD CONSTRAINT ck_flujo_estado CHECK (Estado IN ('ACTIVO','INACTIVO'));

UPDATE TIPO_EVENTO SET Estado = UPPER(Estado);
ALTER TABLE TIPO_EVENTO ALTER COLUMN Estado SET DEFAULT 'ACTIVO';
ALTER TABLE TIPO_EVENTO ADD CONSTRAINT ck_tipo_evento_estado CHECK (Estado IN ('ACTIVO','INACTIVO'));
