-- =====================================================================
-- BEL Comercial R1 · Tablas de relación persona–grupo–empresa
-- Motor: Aurora MySQL (compatible MySQL 8)
-- Incluye los ajustes acordados: contador en varios grupos, administración
-- Dual, enrollment_id, estados con CHECK, Version y correos de notificación
-- por relación en BackOffice.
-- Supone que ya existen: Grupo_economico, Empresas_asociadas, role,
-- Tipo_visualizacion, Tipo_documento.
-- =====================================================================

-- ---------------------------------------------------------------------
-- 0. Llaves compuestas en las tablas padre (para que empresa y rol
--    siempre sean del mismo grupo que la relación)
-- ---------------------------------------------------------------------
ALTER TABLE Empresas_asociadas
  ADD CONSTRAINT uq_empresa_grupo UNIQUE (id_empresa_asociada, id_grupo_economico);

ALTER TABLE role
  ADD CONSTRAINT uq_role_grupo UNIQUE (id_role, id_grupo_economico);

-- ---------------------------------------------------------------------
-- 1. belc_persona_canal · único registro local de la persona
-- ---------------------------------------------------------------------
CREATE TABLE belc_persona_canal (
  Identity_id        VARCHAR(64)  NOT NULL,              -- _id de managed/user en ForgeRock
  Estado_canal       VARCHAR(25)  NOT NULL DEFAULT 'PENDIENTE_HABILITACION',
  Nombre_mostrar     VARCHAR(150) NOT NULL,              -- copia de visualización, se refresca en cada login
  Terms_version      VARCHAR(20)  NOT NULL,
  Terms_accepted_at  DATETIME(3)  NOT NULL,
  Fecha_creacion     DATETIME(3)  NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
  Fecha_modificacion DATETIME(3)  NULL ON UPDATE CURRENT_TIMESTAMP(3),
  Version            INT          NOT NULL DEFAULT 1,
  CONSTRAINT pk_belc_persona_canal PRIMARY KEY (Identity_id),
  CONSTRAINT ck_bpc_estado CHECK (Estado_canal IN ('PENDIENTE_HABILITACION','ACTIVO','BAJA'))
) ENGINE = InnoDB DEFAULT CHARSET = utf8mb4;

-- ---------------------------------------------------------------------
-- 2. afiliacion · una invitación por persona y empresa
-- ---------------------------------------------------------------------
CREATE TABLE afiliacion (
  id_afiliacion          INT          NOT NULL AUTO_INCREMENT,
  enrollment_id          CHAR(36)     NOT NULL,          -- viaja en el enlace; idempotencia del journey
  id_grupo_economico     INT          NOT NULL,
  id_empresa_asociada    INT          NOT NULL,
  id_rol_asignado        INT          NOT NULL,
  id_tipo_visualizacion  INT          NOT NULL,
  id_tipo_documento      INT          NOT NULL,
  Documento              VARCHAR(30)  NOT NULL,          -- normalizado con la regla del nodo 040
  Correo_electronico     VARCHAR(254) NOT NULL,          -- contacto de la invitación (OTP)
  Token_hash             VARBINARY(32) NOT NULL,
  Identity_id            VARCHAR(64)  NULL,              -- se llena en el paso 200 (Vincular persona)
  Estado                 VARCHAR(15)  NOT NULL DEFAULT 'PENDIENTE',
  Fecha_envio            DATETIME(3)  NOT NULL,
  Fecha_expiracion       DATETIME(3)  NOT NULL,
  Fecha_aceptacion       DATETIME(3)  NULL,
  Creado_por             VARCHAR(80)  NOT NULL,
  Fecha_creacion         DATETIME(3)  NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
  Version                INT          NOT NULL DEFAULT 1,
  CONSTRAINT pk_afiliacion PRIMARY KEY (id_afiliacion),
  CONSTRAINT uq_afiliacion_enrollment UNIQUE (enrollment_id),
  CONSTRAINT ck_afiliacion_estado CHECK (Estado IN ('PENDIENTE','INCOMPLETA','COMPLETADA','EXPIRADA','REVOCADA')),
  CONSTRAINT ck_afiliacion_completada CHECK (Estado <> 'COMPLETADA' OR (Identity_id IS NOT NULL AND Fecha_aceptacion IS NOT NULL)),
  CONSTRAINT fk_afiliacion_empresa_grupo FOREIGN KEY (id_empresa_asociada, id_grupo_economico)
    REFERENCES Empresas_asociadas (id_empresa_asociada, id_grupo_economico),
  CONSTRAINT fk_afiliacion_role_grupo FOREIGN KEY (id_rol_asignado, id_grupo_economico)
    REFERENCES role (id_role, id_grupo_economico),
  CONSTRAINT fk_afiliacion_visualizacion FOREIGN KEY (id_tipo_visualizacion)
    REFERENCES Tipo_visualizacion (id_tipo_visualizacion),
  CONSTRAINT fk_afiliacion_tipo_documento FOREIGN KEY (id_tipo_documento)
    REFERENCES Tipo_documento (id_tipo_documento),
  CONSTRAINT fk_afiliacion_persona FOREIGN KEY (Identity_id)
    REFERENCES belc_persona_canal (Identity_id),
  INDEX ix_afiliacion_documento_empresa (Documento, id_empresa_asociada, Estado)
) ENGINE = InnoDB DEFAULT CHARSET = utf8mb4;
-- Regla de aplicación: no crear otra invitación PENDIENTE para el mismo
-- Documento + id_empresa_asociada.

-- ---------------------------------------------------------------------
-- 3. Usuario_grupo_economico · una fila por persona y empresa
-- ---------------------------------------------------------------------
CREATE TABLE Usuario_grupo_economico (
  id_usuario_grupo_economico INT         NOT NULL AUTO_INCREMENT,
  Identity_id                VARCHAR(64) NOT NULL,
  id_grupo_economico         INT         NOT NULL,
  id_empresa_asociada        INT         NOT NULL,
  id_role                    INT         NOT NULL,
  id_tipo_visualizacion      INT         NOT NULL,
  id_afiliacion              INT         NOT NULL,       -- invitación que originó la relación
  Estado                     VARCHAR(10) NOT NULL DEFAULT 'ACTIVO',
  Creado_por                 VARCHAR(80) NOT NULL,
  Fecha_creacion             DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
  Modificado_por             VARCHAR(80) NULL,
  Fecha_modificacion         DATETIME(3) NULL ON UPDATE CURRENT_TIMESTAMP(3),
  Version                    INT         NOT NULL DEFAULT 1,  -- control optimista de aprobaciones
  CONSTRAINT pk_usuario_grupo_economico PRIMARY KEY (id_usuario_grupo_economico),
  CONSTRAINT uq_uge_identity_empresa UNIQUE (Identity_id, id_empresa_asociada),
  CONSTRAINT uq_uge_afiliacion UNIQUE (id_afiliacion),
  CONSTRAINT ck_uge_estado CHECK (Estado IN ('ACTIVO','INACTIVO')),
  CONSTRAINT fk_uge_persona FOREIGN KEY (Identity_id)
    REFERENCES belc_persona_canal (Identity_id),
  CONSTRAINT fk_uge_empresa_grupo FOREIGN KEY (id_empresa_asociada, id_grupo_economico)
    REFERENCES Empresas_asociadas (id_empresa_asociada, id_grupo_economico),
  CONSTRAINT fk_uge_role_grupo FOREIGN KEY (id_role, id_grupo_economico)
    REFERENCES role (id_role, id_grupo_economico),
  CONSTRAINT fk_uge_visualizacion FOREIGN KEY (id_tipo_visualizacion)
    REFERENCES Tipo_visualizacion (id_tipo_visualizacion),
  CONSTRAINT fk_uge_afiliacion FOREIGN KEY (id_afiliacion)
    REFERENCES afiliacion (id_afiliacion),
  INDEX ix_uge_identity_estado (Identity_id, Estado)   -- consulta del login
) ENGINE = InnoDB DEFAULT CHARSET = utf8mb4;

-- ---------------------------------------------------------------------
-- 4. Grupo_economico_administrador · 1 (Individual) o 2 (Dual) por grupo
-- ---------------------------------------------------------------------
CREATE TABLE Grupo_economico_administrador (
  id_grupo_economico_administrador INT         NOT NULL AUTO_INCREMENT,
  id_grupo_economico               INT         NOT NULL,
  Identity_id                      VARCHAR(64) NOT NULL,
  Estado                           VARCHAR(10) NOT NULL DEFAULT 'ACTIVO',
  Creado_por                       VARCHAR(80) NOT NULL,
  Fecha_creacion                   DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
  Version                          INT         NOT NULL DEFAULT 1,
  CONSTRAINT pk_grupo_economico_administrador PRIMARY KEY (id_grupo_economico_administrador),
  CONSTRAINT uq_gea_grupo_identity UNIQUE (id_grupo_economico, Identity_id),
  CONSTRAINT ck_gea_estado CHECK (Estado IN ('ACTIVO','INACTIVO')),
  CONSTRAINT fk_gea_grupo FOREIGN KEY (id_grupo_economico)
    REFERENCES Grupo_economico (id_grupo_economico),
  CONSTRAINT fk_gea_persona FOREIGN KEY (Identity_id)
    REFERENCES belc_persona_canal (Identity_id)
) ENGINE = InnoDB DEFAULT CHARSET = utf8mb4;
-- Máximo de administradores según Grupo_economico.Tipo_administracion: validar en la aplicación.

-- ---------------------------------------------------------------------
-- 5. Contacto_notificacion · correo de negocio por relación
--    Los avisos de seguridad (OTP, recuperación, bloqueo) usan el mail de ForgeRock.
-- ---------------------------------------------------------------------
CREATE TABLE Contacto_notificacion (
  id_contacto_notificacion   BIGINT       NOT NULL AUTO_INCREMENT,
  id_usuario_grupo_economico INT          NOT NULL,
  Correo                     VARCHAR(254) NOT NULL,
  Estado                     VARCHAR(25)  NOT NULL DEFAULT 'PENDIENTE_VERIFICACION',
  Origen                     VARCHAR(15)  NOT NULL,
  id_afiliacion              INT          NULL,
  Verificado_at              DATETIME(3)  NULL,
  Creado_por                 VARCHAR(80)  NOT NULL,
  Fecha_creacion             DATETIME(3)  NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
  Version                    INT          NOT NULL DEFAULT 1,
  relacion_activa INT GENERATED ALWAYS AS
    (CASE WHEN Estado = 'VERIFICADO' THEN id_usuario_grupo_economico END) STORED,
  CONSTRAINT pk_contacto_notificacion PRIMARY KEY (id_contacto_notificacion),
  CONSTRAINT uq_contacto_activo UNIQUE (relacion_activa),   -- un solo correo verificado por relación
  CONSTRAINT ck_contacto_estado CHECK (Estado IN ('PENDIENTE_VERIFICACION','VERIFICADO','INACTIVO')),
  CONSTRAINT ck_contacto_origen CHECK (Origen IN ('INVITACION','USUARIO','ADMIN')),
  CONSTRAINT ck_contacto_verificado CHECK (Estado <> 'VERIFICADO' OR Verificado_at IS NOT NULL),
  CONSTRAINT fk_contacto_uge FOREIGN KEY (id_usuario_grupo_economico)
    REFERENCES Usuario_grupo_economico (id_usuario_grupo_economico),
  CONSTRAINT fk_contacto_afiliacion FOREIGN KEY (id_afiliacion)
    REFERENCES afiliacion (id_afiliacion)
) ENGINE = InnoDB DEFAULT CHARSET = utf8mb4;
