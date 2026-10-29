-- =====================================================================
-- BELC · Solo las 16 tablas corregidas en la revisión de integridad (v2)
-- Aurora MySQL 3 (compatible MySQL 8.0)
--
-- Cambios: FK compuestas para que empresa, rol, productos, flujo y aprobadores
-- pertenezcan al MISMO grupo económico; la decisión transaccional valida nivel,
-- regla y aprobador; la decisión administrativa exige ser administrador del grupo;
-- CHECK (Aprobaciones_requeridas > 0).
--
-- Estas tablas dependen de otras del modelo BELC (Grupo_economico,
-- Empresas_asociadas, perfil, permiso, moneda, etc.). Para una base nueva,
-- usar el script completo. Para una base existente, borrar estas tablas
-- (si no tienen datos) y crearlas de nuevo con este script.
-- =====================================================================

SET NAMES utf8mb4;
SET FOREIGN_KEY_CHECKS = 0;

CREATE TABLE Producto_vinculacion (
  id_producto_vinculado       INT NOT NULL AUTO_INCREMENT,
  id_empresa_asociada         INT NOT NULL,
  id_grupo_economico          INT NOT NULL,   -- [INTEGRIDAD] copiado de la empresa
  Core_module                 VARCHAR(10) NOT NULL,
  Core_product_id             VARCHAR(40) NOT NULL,
  Core_cliente_id             VARCHAR(30),   -- [PENDIENTE] ¿igual a core_empresasoc_id?
  id_tipo_producto            INT NOT NULL,
  -- [CORRECCIÓN] Se quita la columna Moneda; queda solo id_moneda.
  id_moneda                   INT NOT NULL,
  Numero_mascara              VARCHAR(30) NOT NULL,
  Estado_core                 VARCHAR(30),   -- sin CHECK: refleja los valores del core
  Fecha_ultima_sincronizacion DATETIME,
  Estado_Vinculacion          VARCHAR(12) COLLATE utf8mb4_bin NOT NULL DEFAULT 'PENDIENTE',
  Motivo_estado               VARCHAR(255),
  Fecha_vinculacion           DATETIME,
  Fecha_desvinculacion        DATETIME,
  Creado_por                  VARCHAR(80),
  Modificado_por              VARCHAR(80),
  Fecha_modificacion          DATETIME,
  PRIMARY KEY (id_producto_vinculado),
  -- [CORRECCIÓN] Un producto no se vincula dos veces (HU-04).
  CONSTRAINT uq_producto_core UNIQUE (Core_module, Core_product_id),
  CONSTRAINT uq_producto_grupo UNIQUE (id_producto_vinculado, id_grupo_economico),
  CONSTRAINT ck_producto_estado CHECK (Estado_Vinculacion IN ('PENDIENTE','ACTIVO','SUSPENDIDO','DESVINCULADO')),
  CONSTRAINT fk_producto_empresa FOREIGN KEY (id_empresa_asociada, id_grupo_economico)
    REFERENCES Empresas_asociadas (id_empresa_asociada, id_grupo_economico),
  CONSTRAINT fk_producto_tipo    FOREIGN KEY (id_tipo_producto)    REFERENCES Tipo_producto (id_tipo_producto),
  CONSTRAINT fk_producto_moneda  FOREIGN KEY (id_moneda)           REFERENCES moneda (id_moneda)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE Grupo_producto (
  id_Grupo_producto  INT NOT NULL AUTO_INCREMENT,
  id_grupo_economico INT NOT NULL,
  Nombre_grupo       VARCHAR(100) NOT NULL,
  Descripcion        VARCHAR(255),
  Estado             VARCHAR(10) COLLATE utf8mb4_bin NOT NULL DEFAULT 'ACTIVO',
  Version            INT NOT NULL DEFAULT 1,   -- punto 2 de la revisión
  Fecha_creacion     DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  Creado_por         VARCHAR(80),
  Modificado_por     VARCHAR(80),
  Fecha_modificacion DATETIME,
  PRIMARY KEY (id_Grupo_producto),
  CONSTRAINT uq_grupo_producto_nombre UNIQUE (id_grupo_economico, Nombre_grupo),
  CONSTRAINT uq_grupo_producto_grupo UNIQUE (id_Grupo_producto, id_grupo_economico),
  CONSTRAINT ck_grupo_producto_estado CHECK (Estado IN ('ACTIVO','INACTIVO')),
  CONSTRAINT fk_grupoprod_grupo FOREIGN KEY (id_grupo_economico) REFERENCES Grupo_economico (id_grupo_economico)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE Grupo_producto_detalle (
  id_grupo_producto_detalle INT NOT NULL AUTO_INCREMENT,
  id_grupo_producto         INT NOT NULL,
  id_producto_vinculacion   INT NOT NULL,
  id_grupo_economico        INT NOT NULL,   -- [INTEGRIDAD]
  Fecha_creacion            DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  Creado_por                VARCHAR(80),
  PRIMARY KEY (id_grupo_producto_detalle),
  CONSTRAINT uq_grupo_producto_detalle UNIQUE (id_grupo_producto, id_producto_vinculacion),
  -- [INTEGRIDAD] el grupo de productos y el producto son del mismo grupo económico
  CONSTRAINT fk_gpd_grupo    FOREIGN KEY (id_grupo_producto, id_grupo_economico)
    REFERENCES Grupo_producto (id_Grupo_producto, id_grupo_economico),
  CONSTRAINT fk_gpd_producto FOREIGN KEY (id_producto_vinculacion, id_grupo_economico)
    REFERENCES Producto_vinculacion (id_producto_vinculado, id_grupo_economico)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE role (
  id_role            INT NOT NULL AUTO_INCREMENT,
  Codigo_role        VARCHAR(30)  NOT NULL,
  Nombre_role        VARCHAR(100) NOT NULL,
  Descripcion        VARCHAR(255),
  Estado             VARCHAR(10) COLLATE utf8mb4_bin NOT NULL DEFAULT 'ACTIVO',
  id_perfil          INT NOT NULL,
  id_grupo_economico INT NOT NULL,
  Version            INT NOT NULL DEFAULT 1,   -- punto 2 de la revisión
  PRIMARY KEY (id_role),
  CONSTRAINT uq_role_codigo UNIQUE (id_grupo_economico, Codigo_role),
  CONSTRAINT uq_role_grupo UNIQUE (id_role, id_grupo_economico),
  CONSTRAINT ck_role_estado CHECK (Estado IN ('ACTIVO','INACTIVO')),
  CONSTRAINT fk_role_perfil FOREIGN KEY (id_perfil)          REFERENCES perfil (id_perfil),
  CONSTRAINT fk_role_grupo  FOREIGN KEY (id_grupo_economico) REFERENCES Grupo_economico (id_grupo_economico)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE Rol_grupo_producto (
  id_rol_grupo_producto INT NOT NULL AUTO_INCREMENT,
  id_role               INT NOT NULL,
  id_grupo_producto     INT NOT NULL,
  id_grupo_economico    INT NOT NULL,   -- [INTEGRIDAD]
  Fecha_creacion        DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  Creado_por            VARCHAR(80),
  PRIMARY KEY (id_rol_grupo_producto),
  CONSTRAINT uq_rol_grupo_producto UNIQUE (id_role, id_grupo_producto),
  -- [INTEGRIDAD] el rol y el grupo de productos son del mismo grupo económico
  CONSTRAINT fk_rgp_role  FOREIGN KEY (id_role, id_grupo_economico)
    REFERENCES role (id_role, id_grupo_economico),
  CONSTRAINT fk_rgp_grupo FOREIGN KEY (id_grupo_producto, id_grupo_economico)
    REFERENCES Grupo_producto (id_Grupo_producto, id_grupo_economico)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE afiliacion (
  id_afiliacion         INT NOT NULL AUTO_INCREMENT,   -- = enrollment_id
  id_grupo_economico    INT NOT NULL,
  id_empresa_asociada   INT,
  id_rol_asignado       INT NOT NULL,
  Identity_id           VARCHAR(64),                   -- se llena al aceptar
  id_tipo_visualizacion INT,
  -- [CORRECCIÓN] Nombre agregado (para listar invitaciones pendientes, HU-07).
  Nombre                VARCHAR(150) NOT NULL,
  Correo_electronico    VARCHAR(150) NOT NULL,
  id_tipo_documento     INT NOT NULL,
  Documento             VARCHAR(30)  NOT NULL,
  Token_hash            CHAR(64) NOT NULL,             -- SHA-256 del token secreto
  Estado                VARCHAR(12) COLLATE utf8mb4_bin NOT NULL DEFAULT 'PENDIENTE',
  Fecha_envio           DATETIME NOT NULL,
  Fecha_expiracion      DATETIME NOT NULL,
  Fecha_aceptacion      DATETIME NULL,
  Creado_por            VARCHAR(80),
  Fecha_creacion        DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (id_afiliacion),
  CONSTRAINT uq_afiliacion_token UNIQUE (Token_hash),
  CONSTRAINT ck_afiliacion_estado CHECK (Estado IN ('PENDIENTE','COMPLETADA','INCOMPLETA','EXPIRADA','REVOCADA')),
  CONSTRAINT fk_af_grupo   FOREIGN KEY (id_grupo_economico)    REFERENCES Grupo_economico (id_grupo_economico),
  -- [INTEGRIDAD] la empresa y el rol son del mismo grupo de la afiliación
  CONSTRAINT fk_af_empresa FOREIGN KEY (id_empresa_asociada, id_grupo_economico)
    REFERENCES Empresas_asociadas (id_empresa_asociada, id_grupo_economico),
  CONSTRAINT fk_af_rol     FOREIGN KEY (id_rol_asignado, id_grupo_economico)
    REFERENCES role (id_role, id_grupo_economico),
  CONSTRAINT fk_af_persona FOREIGN KEY (Identity_id)           REFERENCES Belc_persona_canal (Identity_id),
  CONSTRAINT fk_af_visual  FOREIGN KEY (id_tipo_visualizacion) REFERENCES Tipo_visualizacion (id_tipo_visualizacion),
  CONSTRAINT fk_af_tipodoc FOREIGN KEY (id_tipo_documento)     REFERENCES Tipo_documento (id_tipo_documento)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
CREATE INDEX ix_afiliacion_grupo_estado ON afiliacion (id_grupo_economico, Estado);

CREATE TABLE Usuario_grupo_economico (
  id_usuario_grupo_economico INT NOT NULL AUTO_INCREMENT,
  id_grupo_economico         INT NOT NULL,
  id_empresa_asociada        INT,
  Identity_id                VARCHAR(64) NOT NULL,
  id_role                    INT NOT NULL,
  id_tipo_visualizacion      INT,
  id_afiliacion              INT,
  Estado                     VARCHAR(10) COLLATE utf8mb4_bin NOT NULL DEFAULT 'ACTIVO',
  Fecha_creacion             DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  Creado_por                 VARCHAR(80),
  Modificado_por             VARCHAR(80),
  PRIMARY KEY (id_usuario_grupo_economico),
  -- [PENDIENTE] ¿La relación es por empresa o por grupo económico? Se deja como el diagrama:
  -- UNIQUE (Identity_id, id_empresa_asociada). Si se decide por grupo, cambiar a
  -- UNIQUE (Identity_id, id_grupo_economico) y quitar id_empresa_asociada.
  CONSTRAINT uq_persona_empresa UNIQUE (Identity_id, id_empresa_asociada),
  CONSTRAINT uq_uge_grupo UNIQUE (id_usuario_grupo_economico, id_grupo_economico),
  CONSTRAINT ck_uge_estado CHECK (Estado IN ('ACTIVO','INACTIVO')),
  CONSTRAINT fk_uge_grupo   FOREIGN KEY (id_grupo_economico)    REFERENCES Grupo_economico (id_grupo_economico),
  -- [INTEGRIDAD] la empresa y el rol son del mismo grupo de la relación
  CONSTRAINT fk_uge_empresa FOREIGN KEY (id_empresa_asociada, id_grupo_economico)
    REFERENCES Empresas_asociadas (id_empresa_asociada, id_grupo_economico),
  CONSTRAINT fk_uge_persona FOREIGN KEY (Identity_id)           REFERENCES Belc_persona_canal (Identity_id),
  CONSTRAINT fk_uge_role    FOREIGN KEY (id_role, id_grupo_economico)
    REFERENCES role (id_role, id_grupo_economico),
  CONSTRAINT fk_uge_visual  FOREIGN KEY (id_tipo_visualizacion) REFERENCES Tipo_visualizacion (id_tipo_visualizacion),
  CONSTRAINT fk_uge_af      FOREIGN KEY (id_afiliacion)         REFERENCES afiliacion (id_afiliacion)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
CREATE INDEX ix_uge_grupo_estado ON Usuario_grupo_economico (id_grupo_economico, Estado);

CREATE TABLE Flujo_aprobacion (
  id_flujo           INT NOT NULL AUTO_INCREMENT,
  id_grupo_economico INT NOT NULL,
  Codigo             VARCHAR(30)  NOT NULL,
  Nombre             VARCHAR(150) NOT NULL,
  Version            INT NOT NULL DEFAULT 1,
  Estado             VARCHAR(10) COLLATE utf8mb4_bin NOT NULL DEFAULT 'ACTIVO',
  Fecha_creacion     DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  Creado_por         VARCHAR(80),
  PRIMARY KEY (id_flujo),
  CONSTRAINT uq_flujo_codigo UNIQUE (id_grupo_economico, Codigo),
  CONSTRAINT uq_flujo_grupo UNIQUE (id_flujo, id_grupo_economico),
  CONSTRAINT ck_flujo_estado CHECK (Estado IN ('ACTIVO','INACTIVO')),
  CONSTRAINT fk_flujo_grupo FOREIGN KEY (id_grupo_economico) REFERENCES Grupo_economico (id_grupo_economico)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE Flujo_aprobacion_regla (
  id_regla            INT NOT NULL AUTO_INCREMENT,
  id_flujo            INT NOT NULL,
  id_grupo_economico  INT NOT NULL,   -- [INTEGRIDAD]
  id_tipo_transaccion INT NOT NULL,
  -- [CORRECCIÓN] Se quita la columna Moneda; queda solo id_moneda.
  id_moneda           INT NOT NULL,
  Monto_desde         DECIMAL(18,2) NOT NULL,
  Monto_hasta         DECIMAL(18,2) NOT NULL,
  PRIMARY KEY (id_regla),
  CONSTRAINT uq_regla_grupo UNIQUE (id_regla, id_grupo_economico),
  CONSTRAINT uq_regla_flujo UNIQUE (id_regla, id_flujo),
  -- Monto_desde inclusivo, Monto_hasta exclusivo. Los solapamientos los valida el servicio.
  CONSTRAINT ck_regla_montos CHECK (Monto_desde >= 0 AND Monto_hasta > Monto_desde),
  CONSTRAINT fk_regla_flujo  FOREIGN KEY (id_flujo, id_grupo_economico)
    REFERENCES Flujo_aprobacion (id_flujo, id_grupo_economico),
  CONSTRAINT fk_regla_tipotx FOREIGN KEY (id_tipo_transaccion) REFERENCES Tipo_transaccion (id_tipo_transaccion),
  CONSTRAINT fk_regla_moneda FOREIGN KEY (id_moneda)           REFERENCES moneda (id_moneda)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE Flujo_aprobacion_nivel (
  id_nivel                INT NOT NULL AUTO_INCREMENT,
  id_regla                INT NOT NULL,
  id_grupo_economico      INT NOT NULL,   -- [INTEGRIDAD]
  Orden                   INT NOT NULL,
  Aprobaciones_requeridas INT NOT NULL,
  PRIMARY KEY (id_nivel),
  CONSTRAINT uq_nivel_orden UNIQUE (id_regla, Orden),
  CONSTRAINT uq_nivel_regla UNIQUE (id_nivel, id_regla),
  CONSTRAINT uq_nivel_grupo UNIQUE (id_nivel, id_grupo_economico),
  CONSTRAINT ck_nivel_aprob CHECK (Aprobaciones_requeridas > 0),
  CONSTRAINT fk_nivel_regla FOREIGN KEY (id_regla, id_grupo_economico)
    REFERENCES Flujo_aprobacion_regla (id_regla, id_grupo_economico)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE Flujo_aprobacion_aprobador (
  id_Flujo_aprobacion_aprobador INT NOT NULL AUTO_INCREMENT,
  id_nivel                      INT NOT NULL,
  id_usuario_grupo_economico    INT NOT NULL,
  id_grupo_economico            INT NOT NULL,   -- [INTEGRIDAD]
  PRIMARY KEY (id_Flujo_aprobacion_aprobador),
  CONSTRAINT uq_aprobador UNIQUE (id_nivel, id_usuario_grupo_economico),
  -- [INTEGRIDAD] el aprobador es del mismo grupo que el flujo
  CONSTRAINT fk_aprobador_nivel FOREIGN KEY (id_nivel, id_grupo_economico)
    REFERENCES Flujo_aprobacion_nivel (id_nivel, id_grupo_economico),
  CONSTRAINT fk_aprobador_uge   FOREIGN KEY (id_usuario_grupo_economico, id_grupo_economico)
    REFERENCES Usuario_grupo_economico (id_usuario_grupo_economico, id_grupo_economico)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE Flujo_aprobacion_producto (
  id_flujo_producto       INT NOT NULL AUTO_INCREMENT,
  id_flujo                INT NOT NULL,
  id_producto_vinculacion INT NOT NULL,
  id_grupo_economico      INT NOT NULL,   -- [INTEGRIDAD]
  PRIMARY KEY (id_flujo_producto),
  CONSTRAINT uq_flujo_producto UNIQUE (id_flujo, id_producto_vinculacion),
  -- [INTEGRIDAD] el flujo y el producto son del mismo grupo económico
  CONSTRAINT fk_fp_flujo    FOREIGN KEY (id_flujo, id_grupo_economico)
    REFERENCES Flujo_aprobacion (id_flujo, id_grupo_economico),
  CONSTRAINT fk_fp_producto FOREIGN KEY (id_producto_vinculacion, id_grupo_economico)
    REFERENCES Producto_vinculacion (id_producto_vinculado, id_grupo_economico)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE Aprobacion_administrativa (
  id_aprobacion           INT NOT NULL AUTO_INCREMENT,
  id_grupo_economico      INT NOT NULL,
  -- [CORRECCIÓN] Se quita la columna Tipo_cambio: era redundante con id_tipo_cambio.
  id_tipo_cambio          INT NOT NULL,
  Entidad                 VARCHAR(50) NOT NULL,
  id_registro             VARCHAR(50) NOT NULL,
  Accion                  VARCHAR(20) NOT NULL,
  Datos_anteriores        JSON,
  Datos_propuestos        JSON,
  Version_registro        INT NOT NULL,
  Aprobaciones_requeridas INT NOT NULL DEFAULT 1,
  Estado                  VARCHAR(10) COLLATE utf8mb4_bin NOT NULL DEFAULT 'PENDIENTE',
  Fecha_solicitud         DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  Fecha_expiracion        DATETIME NOT NULL,
  Solicitado_por          VARCHAR(64) NOT NULL,
  Motivo_fallo            VARCHAR(255),
  id_correlacion          VARCHAR(64),
  PRIMARY KEY (id_aprobacion),
  CONSTRAINT uq_aprobadm_grupo UNIQUE (id_aprobacion, id_grupo_economico),
  CONSTRAINT ck_aprobadm_estado CHECK (Estado IN ('PENDIENTE','APROBADA','APLICADA','RECHAZADA','EXPIRADA','CANCELADA','FALLIDA')),
  -- [CORRECCIÓN v2]
  CONSTRAINT ck_aprobadm_requeridas CHECK (Aprobaciones_requeridas > 0),
  CONSTRAINT fk_aprobadm_grupo FOREIGN KEY (id_grupo_economico) REFERENCES Grupo_economico (id_grupo_economico),
  CONSTRAINT fk_aprobadm_tipo  FOREIGN KEY (id_tipo_cambio)     REFERENCES Tipo_cambio_administrativo (id_tipo_cambio)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE Aprobacion_administrativa_decision (
  id_decision        INT NOT NULL AUTO_INCREMENT,
  id_aprobacion      INT NOT NULL,
  id_grupo_economico INT NOT NULL,   -- [INTEGRIDAD]
  Identity_id        VARCHAR(64) NOT NULL,
  Decision           VARCHAR(10) COLLATE utf8mb4_bin NOT NULL,
  Comentario         VARCHAR(400),
  Fecha_decision_utc DATETIME(3) NOT NULL,
  PRIMARY KEY (id_decision),
  CONSTRAINT uq_aprobadm_decision UNIQUE (id_aprobacion, Identity_id),
  CONSTRAINT ck_aprobadm_decision CHECK (Decision IN ('APROBADA','RECHAZADA')),
  CONSTRAINT fk_aprobadmdec_aprob   FOREIGN KEY (id_aprobacion, id_grupo_economico)
    REFERENCES Aprobacion_administrativa (id_aprobacion, id_grupo_economico),
  -- [INTEGRIDAD] solo decide un administrador de ese grupo (que esté ACTIVO lo valida el servicio)
  CONSTRAINT fk_aprobadmdec_admin   FOREIGN KEY (id_grupo_economico, Identity_id)
    REFERENCES Grupo_economico_administrador (id_grupo_economico, Identity_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE Transaccion_aprobacion (
  id_transaccion_aprobacion INT NOT NULL AUTO_INCREMENT,
  id_grupo_economico INT NOT NULL,
  id_flujo           INT NOT NULL,
  Version_flujo      INT NOT NULL,
  id_regla           INT NOT NULL,
  Creado_por         INT NOT NULL,   -- id_usuario_grupo_economico
  Monto              DECIMAL(18,2) NOT NULL,
  id_moneda          INT NOT NULL,
  Cuenta_origen      VARCHAR(40) NOT NULL,
  Cuenta_destino     VARCHAR(40) NOT NULL,
  Estado             VARCHAR(15) COLLATE utf8mb4_bin NOT NULL DEFAULT 'PENDIENTE',
  Nivel_actual       INT NULL,
  Fecha_creacion     DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  Fecha_expiracion   DATETIME NOT NULL,
  Referencia_core    VARCHAR(40) NULL,
  Version            INT NOT NULL DEFAULT 1,
  PRIMARY KEY (id_transaccion_aprobacion),
  CONSTRAINT ck_trx_estado CHECK (Estado IN ('PENDIENTE','EN_APROBACION','APROBADA','EJECUTADA',
                                             'RECHAZADA','EXPIRADA','CANCELADA','FALLIDA')),
  CONSTRAINT ck_trx_monto  CHECK (Monto > 0),
  CONSTRAINT uq_trx_regla  UNIQUE (id_transaccion_aprobacion, id_regla),
  CONSTRAINT fk_trx_grupo   FOREIGN KEY (id_grupo_economico) REFERENCES Grupo_economico (id_grupo_economico),
  -- [INTEGRIDAD] flujo y creador del mismo grupo; la regla es del flujo
  CONSTRAINT fk_trx_flujo   FOREIGN KEY (id_flujo, id_grupo_economico)
    REFERENCES Flujo_aprobacion (id_flujo, id_grupo_economico),
  CONSTRAINT fk_trx_regla   FOREIGN KEY (id_regla, id_flujo)
    REFERENCES Flujo_aprobacion_regla (id_regla, id_flujo),
  CONSTRAINT fk_trx_creador FOREIGN KEY (Creado_por, id_grupo_economico)
    REFERENCES Usuario_grupo_economico (id_usuario_grupo_economico, id_grupo_economico),
  CONSTRAINT fk_trx_moneda  FOREIGN KEY (id_moneda)          REFERENCES moneda (id_moneda)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE Transaccion_aprobacion_decision (
  id_decision                INT NOT NULL AUTO_INCREMENT,
  id_transaccion_aprobacion  INT NOT NULL,
  id_regla                   INT NOT NULL,   -- [INTEGRIDAD]
  id_nivel                   INT NOT NULL,
  id_usuario_grupo_economico INT NOT NULL,
  Decision                   VARCHAR(10) COLLATE utf8mb4_bin NOT NULL,
  Comentario                 VARCHAR(400) NULL,
  Fecha_decision             DATETIME NOT NULL,
  PRIMARY KEY (id_decision),
  CONSTRAINT uq_trxdecision_una_vez UNIQUE (id_transaccion_aprobacion, id_usuario_grupo_economico),
  CONSTRAINT ck_trxdecision CHECK (Decision IN ('APROBADA','RECHAZADA')),
  -- [INTEGRIDAD] el nivel es de la regla de la transacción y quien firma es aprobador de ese nivel.
  -- Que el creador no se apruebe a sí mismo lo valida el servicio (decisión de la PO).
  CONSTRAINT fk_trxdec_trx   FOREIGN KEY (id_transaccion_aprobacion, id_regla)
    REFERENCES Transaccion_aprobacion (id_transaccion_aprobacion, id_regla),
  CONSTRAINT fk_trxdec_nivel FOREIGN KEY (id_nivel, id_regla)
    REFERENCES Flujo_aprobacion_nivel (id_nivel, id_regla),
  CONSTRAINT fk_trxdec_aprobador FOREIGN KEY (id_nivel, id_usuario_grupo_economico)
    REFERENCES Flujo_aprobacion_aprobador (id_nivel, id_usuario_grupo_economico)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

SET FOREIGN_KEY_CHECKS = 1;
