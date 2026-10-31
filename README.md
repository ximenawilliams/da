
--
-- Convención de nombres (decisión de arquitectura / DBA):
--   * Prefijo belpc_ en todas las tablas de banca comercial: soporte identifica
--     el sistema por el nombre, aunque se omita el esquema. (bc_ es de otro sistema.)
--   * Inglés, snake_case. PK = <entidad>_id; la FK se llama igual que la PK que
--     referencia (con prefijo de rol cuando aplica, ej. assigned_role_id).
--   * Fechas *_at · booleanos is_* · auditoría created_at, created_by, updated_at,
--     updated_by, version.
--   * person_id = id de la persona (_id de ForgeRock); es la llave que relaciona con BackOffice.
--     No hay tabla maestra de personas: belpc_person_channel es su registro en el canal
--     (excepción a la regla <entidad>_id: su PK es person_id).
--   * Constraints con nombre explicativo: uq_<tabla>__<columnas>, fk_<tabla>__<tabla_padre>,
--     ck_<tabla>__<columna>, ix_<tabla>__<columnas>. El error dice qué regla falló.
--   * Los VALORES de estados y códigos (ACTIVO, PENDIENTE, CREADOR...) no cambian.
--   * Cada tabla tiene COMMENT con su dominio y para qué sirve.
--   * Equivalencias con los nombres anteriores: Diccionario_nombres_BELC_v4.xlsx
-- =====================================================================

SET NAMES utf8mb4;

-- =====================================================================
-- Catálogos
-- =====================================================================
-- Tipos de documento de identidad (cédula, pasaporte).
CREATE TABLE belpc_document_type (
  document_type_id INT NOT NULL AUTO_INCREMENT,
  code VARCHAR(20) NOT NULL,
  name VARCHAR(100) NOT NULL,
  status VARCHAR(10) COLLATE utf8mb4_bin NOT NULL DEFAULT 'ACTIVO',
  PRIMARY KEY (document_type_id),
  CONSTRAINT uq_document_type__code UNIQUE (code),
  CONSTRAINT ck_document_type__status CHECK (status in ('ACTIVO','INACTIVO'))
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
  COMMENT='Catálogos · Tipos de documento de identidad (cédula, pasaporte).';

-- Monedas permitidas.
CREATE TABLE belpc_currency (
  currency_id INT NOT NULL AUTO_INCREMENT,
  code CHAR(3) NOT NULL,
  name VARCHAR(50) NOT NULL,
  status VARCHAR(10) COLLATE utf8mb4_bin NOT NULL DEFAULT 'ACTIVO',
  PRIMARY KEY (currency_id),
  CONSTRAINT uq_currency__code UNIQUE (code),
  CONSTRAINT ck_currency__status CHECK (status in ('ACTIVO','INACTIVO'))
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
  COMMENT='Catálogos · Monedas permitidas.';

-- Cómo ve el usuario sus productos: mixto o separado por empresa.
CREATE TABLE belpc_display_type (
  display_type_id INT NOT NULL AUTO_INCREMENT,
  code VARCHAR(20) NOT NULL,
  name VARCHAR(100) NOT NULL,
  description VARCHAR(255) NULL,
  status VARCHAR(10) COLLATE utf8mb4_bin NOT NULL DEFAULT 'ACTIVO',
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  created_by VARCHAR(80) NULL,
  PRIMARY KEY (display_type_id),
  CONSTRAINT uq_display_type__code UNIQUE (code),
  CONSTRAINT ck_display_type__status CHECK (status in ('ACTIVO','INACTIVO'))
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
  COMMENT='Catálogos · Cómo ve el usuario sus productos: mixto o separado por empresa.';

-- Tipos de producto del core (cuenta corriente, ahorro, préstamo...).
CREATE TABLE belpc_product_type (
  product_type_id INT NOT NULL AUTO_INCREMENT,
  code VARCHAR(30) NOT NULL,
  name VARCHAR(100) NOT NULL,
  is_transactional TINYINT(1) NOT NULL DEFAULT 0,
  masked_number VARCHAR(30) NULL,
  associated_at DATETIME NULL,
  status VARCHAR(10) COLLATE utf8mb4_bin NOT NULL DEFAULT 'ACTIVO',
  created_by VARCHAR(80) NULL,
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (product_type_id),
  CONSTRAINT uq_product_type__code UNIQUE (code),
  CONSTRAINT ck_product_type__status CHECK (status in ('ACTIVO','INACTIVO'))
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
  COMMENT='Catálogos · Tipos de producto del core (cuenta corriente, ahorro, préstamo...).';

-- Tipos de cambio administrativo que pueden requerir aprobación.
CREATE TABLE belpc_admin_change_type (
  admin_change_type_id INT NOT NULL AUTO_INCREMENT,
  code VARCHAR(40) NOT NULL,
  description VARCHAR(255) NULL,
  status VARCHAR(10) COLLATE utf8mb4_bin NOT NULL DEFAULT 'ACTIVO',
  PRIMARY KEY (admin_change_type_id),
  CONSTRAINT uq_admin_change_type__code UNIQUE (code),
  CONSTRAINT ck_admin_change_type__status CHECK (status in ('ACTIVO','INACTIVO'))
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
  COMMENT='Catálogos · Tipos de cambio administrativo que pueden requerir aprobación.';

-- =====================================================================
-- Roles y permisos
-- =====================================================================
-- Permisos del canal comercial (administrativos y monetarios).
CREATE TABLE belpc_permission (
  permission_id INT NOT NULL AUTO_INCREMENT,
  code VARCHAR(60) NOT NULL,
  name VARCHAR(150) NOT NULL,
  category VARCHAR(20) COLLATE utf8mb4_bin NOT NULL,
  status VARCHAR(10) COLLATE utf8mb4_bin NOT NULL DEFAULT 'ACTIVO',
  PRIMARY KEY (permission_id),
  CONSTRAINT uq_permission__code UNIQUE (code),
  CONSTRAINT ck_permission__category CHECK (category in ('ADMINISTRATIVO','MONETARIO')),
  CONSTRAINT ck_permission__status CHECK (status in ('ACTIVO','INACTIVO'))
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
  COMMENT='Roles y permisos · Permisos del canal comercial (administrativos y monetarios).';

-- =====================================================================
-- Catálogos
-- =====================================================================
-- Tipos de transacción monetaria y el permiso que la aprueba.
CREATE TABLE belpc_transaction_type (
  transaction_type_id INT NOT NULL AUTO_INCREMENT,
  code VARCHAR(40) NOT NULL,
  name VARCHAR(150) NOT NULL,
  approver_permission_id INT NULL,
  status VARCHAR(10) COLLATE utf8mb4_bin NOT NULL DEFAULT 'ACTIVO',
  PRIMARY KEY (transaction_type_id),
  CONSTRAINT uq_transaction_type__code UNIQUE (code),
  CONSTRAINT ck_transaction_type__status CHECK (status in ('ACTIVO','INACTIVO')),
  CONSTRAINT fk_transaction_type__permission FOREIGN KEY (approver_permission_id)
    REFERENCES belpc_permission (permission_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
  COMMENT='Catálogos · Tipos de transacción monetaria y el permiso que la aprueba.';

-- =====================================================================
-- Roles y permisos
-- =====================================================================
-- Perfiles base: CREADOR, APROBADOR, ADMINISTRADOR.
CREATE TABLE belpc_profile (
  profile_id INT NOT NULL AUTO_INCREMENT,
  code VARCHAR(30) NOT NULL,
  name VARCHAR(100) NOT NULL,
  description VARCHAR(255) NULL,
  status VARCHAR(10) COLLATE utf8mb4_bin NOT NULL DEFAULT 'ACTIVO',
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  created_by VARCHAR(80) NULL,
  PRIMARY KEY (profile_id),
  CONSTRAINT uq_profile__code UNIQUE (code),
  CONSTRAINT ck_profile__status CHECK (status in ('ACTIVO','INACTIVO'))
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
  COMMENT='Roles y permisos · Perfiles base: CREADOR, APROBADOR, ADMINISTRADOR.';

-- Permisos que cada perfil puede otorgar a un rol.
CREATE TABLE belpc_profile_permission (
  profile_permission_id INT NOT NULL AUTO_INCREMENT,
  permission_id INT NOT NULL,
  profile_id INT NOT NULL,
  PRIMARY KEY (profile_permission_id),
  CONSTRAINT uq_profile_permission__profile_id_permission_id UNIQUE (profile_id, permission_id),
  CONSTRAINT fk_profile_permission__profile FOREIGN KEY (profile_id)
    REFERENCES belpc_profile (profile_id),
  CONSTRAINT fk_profile_permission__permission FOREIGN KEY (permission_id)
    REFERENCES belpc_permission (permission_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
  COMMENT='Roles y permisos · Permisos que cada perfil puede otorgar a un rol.';

-- =====================================================================
-- Grupo económico y productos
-- =====================================================================
-- Grupo económico del cliente comercial. Tipo de administración I (individual) o D (dual).
CREATE TABLE belpc_economic_group (
  economic_group_id INT NOT NULL AUTO_INCREMENT,
  name VARCHAR(100) NOT NULL,
  linked_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  administration_type CHAR(1) COLLATE utf8mb4_bin NOT NULL,
  PRIMARY KEY (economic_group_id),
  CONSTRAINT uq_economic_group__name UNIQUE (name),
  CONSTRAINT ck_economic_group__administration_type CHECK (administration_type in ('I','D'))
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
  COMMENT='Grupo económico y productos · Grupo económico del cliente comercial. Tipo de administración I (individual) o D (dual).';

-- Empresas del core vinculadas a un grupo. Una empresa solo puede estar en un grupo.
CREATE TABLE belpc_company (
  company_id INT NOT NULL AUTO_INCREMENT,
  economic_group_id INT NOT NULL,
  name VARCHAR(150) NOT NULL,
  core_company_id VARCHAR(30) NOT NULL,
  status VARCHAR(12) COLLATE utf8mb4_bin NOT NULL DEFAULT 'ACTIVO',
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  created_by VARCHAR(80) NULL,
  PRIMARY KEY (company_id),
  CONSTRAINT uq_company__core_company_id UNIQUE (core_company_id),
  CONSTRAINT uq_company__company_id_economic_group_id UNIQUE (company_id, economic_group_id),
  CONSTRAINT ck_company__status CHECK (status in ('ACTIVO','DESVINCULADO')),
  CONSTRAINT fk_company__economic_group FOREIGN KEY (economic_group_id)
    REFERENCES belpc_economic_group (economic_group_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
  COMMENT='Grupo económico y productos · Empresas del core vinculadas a un grupo. Una empresa solo puede estar en un grupo.';

-- Productos del core vinculados a una empresa del grupo (número siempre enmascarado).
CREATE TABLE belpc_linked_product (
  linked_product_id INT NOT NULL AUTO_INCREMENT,
  company_id INT NOT NULL,
  economic_group_id INT NOT NULL,
  core_module VARCHAR(10) NOT NULL,
  core_product_id VARCHAR(40) NOT NULL,
  product_type_id INT NOT NULL,
  currency_id INT NOT NULL,
  masked_number VARCHAR(30) NOT NULL,
  core_status VARCHAR(30) NULL,
  last_synced_at DATETIME NULL,
  link_status VARCHAR(12) COLLATE utf8mb4_bin NOT NULL DEFAULT 'PENDIENTE',
  status_reason VARCHAR(255) NULL,
  linked_at DATETIME NULL,
  unlinked_at DATETIME NULL,
  created_by VARCHAR(80) NULL,
  updated_by VARCHAR(80) NULL,
  updated_at DATETIME NULL,
  PRIMARY KEY (linked_product_id),
  CONSTRAINT uq_linked_product__core_module_core_product_id UNIQUE (core_module, core_product_id),
  CONSTRAINT uq_linked_product__linked_product_id_economic_group_id UNIQUE (linked_product_id, economic_group_id),
  CONSTRAINT ck_linked_product__link_status CHECK (link_status in ('PENDIENTE','ACTIVO','SUSPENDIDO','DESVINCULADO')),
  CONSTRAINT fk_linked_product__company FOREIGN KEY (company_id, economic_group_id)
    REFERENCES belpc_company (company_id, economic_group_id),
  CONSTRAINT fk_linked_product__currency FOREIGN KEY (currency_id)
    REFERENCES belpc_currency (currency_id),
  CONSTRAINT fk_linked_product__product_type FOREIGN KEY (product_type_id)
    REFERENCES belpc_product_type (product_type_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
  COMMENT='Grupo económico y productos · Productos del core vinculados a una empresa del grupo (número siempre enmascarado).';

-- Agrupación de productos que se asigna a roles.
CREATE TABLE belpc_product_group (
  product_group_id INT NOT NULL AUTO_INCREMENT,
  economic_group_id INT NOT NULL,
  name VARCHAR(100) NOT NULL,
  description VARCHAR(255) NULL,
  status VARCHAR(10) COLLATE utf8mb4_bin NOT NULL DEFAULT 'ACTIVO',
  is_system TINYINT(1) NOT NULL DEFAULT 0,
  version INT NOT NULL DEFAULT 1,
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  created_by VARCHAR(80) NULL,
  updated_by VARCHAR(80) NULL,
  updated_at DATETIME NULL,
  system_product_group_key INT
    GENERATED ALWAYS AS (case when is_system = 1 then economic_group_id end) STORED,
  PRIMARY KEY (product_group_id),
  CONSTRAINT uq_product_group__product_group_id_economic_group_id UNIQUE (product_group_id, economic_group_id),
  CONSTRAINT uq_product_group__economic_group_id_name UNIQUE (economic_group_id, name),
  CONSTRAINT uq_product_group__system_product_group_key UNIQUE (system_product_group_key),
  CONSTRAINT ck_product_group__status CHECK (status in ('ACTIVO','INACTIVO')),
  CONSTRAINT ck_product_group__is_system CHECK (is_system in (0,1)),
  CONSTRAINT fk_product_group__economic_group FOREIGN KEY (economic_group_id)
    REFERENCES belpc_economic_group (economic_group_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
  COMMENT='Grupo económico y productos · Agrupación de productos que se asigna a roles.';

-- Productos que forman cada grupo de productos.
CREATE TABLE belpc_product_group_item (
  product_group_item_id INT NOT NULL AUTO_INCREMENT,
  product_group_id INT NOT NULL,
  linked_product_id INT NOT NULL,
  economic_group_id INT NOT NULL,
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  created_by VARCHAR(80) NULL,
  PRIMARY KEY (product_group_item_id),
  CONSTRAINT uq_product_group_item__product_group_id_linked_product_id UNIQUE (product_group_id, linked_product_id),
  CONSTRAINT fk_product_group_item__product_group FOREIGN KEY (product_group_id, economic_group_id)
    REFERENCES belpc_product_group (product_group_id, economic_group_id),
  CONSTRAINT fk_product_group_item__linked_product FOREIGN KEY (linked_product_id, economic_group_id)
    REFERENCES belpc_linked_product (linked_product_id, economic_group_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
  COMMENT='Grupo económico y productos · Productos que forman cada grupo de productos.';

-- =====================================================================
-- Personas y accesos
-- =====================================================================
-- Registro de la persona en el canal comercial: estado en el canal, nombre a mostrar y términos aceptados. No es un maestro de personas: la persona vive en ForgeRock/BackOffice; person_id es su id.
CREATE TABLE belpc_person_channel (
  person_id VARCHAR(64) NOT NULL COMMENT 'Id de la persona (_id de ForgeRock). Relaciona con BackOffice.',
  channel_status VARCHAR(10) COLLATE utf8mb4_bin NOT NULL DEFAULT 'ACTIVO',
  display_name VARCHAR(150) NOT NULL,
  terms_version VARCHAR(10) NOT NULL,
  terms_accepted_at DATETIME NOT NULL,
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (person_id),
  CONSTRAINT ck_person_channel__channel_status CHECK (channel_status in ('ACTIVO','INACTIVO'))
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
  COMMENT='Personas y accesos · Registro de la persona en el canal comercial: estado en el canal, nombre a mostrar y términos aceptados. No es un maestro de personas: la persona vive en ForgeRock/BackOffice; person_id es su id.';

-- =====================================================================
-- Roles y permisos
-- =====================================================================
-- Roles del grupo económico. Un solo rol de sistema (ADMINISTRADOR) por grupo.
CREATE TABLE belpc_role (
  role_id INT NOT NULL AUTO_INCREMENT,
  code VARCHAR(30) NOT NULL,
  name VARCHAR(100) NOT NULL,
  description VARCHAR(255) NULL,
  status VARCHAR(10) COLLATE utf8mb4_bin NOT NULL DEFAULT 'ACTIVO',
  is_system TINYINT(1) NOT NULL DEFAULT 0,
  profile_id INT NOT NULL,
  economic_group_id INT NOT NULL,
  version INT NOT NULL DEFAULT 1,
  system_role_group_key INT
    GENERATED ALWAYS AS (case when is_system = 1 then economic_group_id end) STORED,
  PRIMARY KEY (role_id),
  CONSTRAINT uq_role__economic_group_id_code UNIQUE (economic_group_id, code),
  CONSTRAINT uq_role__role_id_economic_group_id UNIQUE (role_id, economic_group_id),
  CONSTRAINT uq_role__role_id_profile_id UNIQUE (role_id, profile_id),
  CONSTRAINT uq_role__system_role_group_key UNIQUE (system_role_group_key),
  CONSTRAINT ck_role__status CHECK (status in ('ACTIVO','INACTIVO')),
  CONSTRAINT ck_role__is_system CHECK (is_system in (0,1)),
  CONSTRAINT fk_role__economic_group FOREIGN KEY (economic_group_id)
    REFERENCES belpc_economic_group (economic_group_id),
  CONSTRAINT fk_role__profile FOREIGN KEY (profile_id)
    REFERENCES belpc_profile (profile_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
  COMMENT='Roles y permisos · Roles del grupo económico. Un solo rol de sistema (ADMINISTRADOR) por grupo.';

-- =====================================================================
-- Personas y accesos
-- =====================================================================
-- Invitaciones (enrollment) a usuarios y administradores. Solo guarda el hash del token.
CREATE TABLE belpc_enrollment (
  enrollment_id INT NOT NULL AUTO_INCREMENT,
  economic_group_id INT NOT NULL,
  company_id INT NULL,
  assigned_role_id INT NOT NULL,
  enrollment_type VARCHAR(15) COLLATE utf8mb4_bin NOT NULL DEFAULT 'USUARIO',
  person_id VARCHAR(64) NULL COMMENT 'Id de la persona (_id de ForgeRock). Relaciona con BackOffice.',
  display_type_id INT NULL,
  name VARCHAR(150) NOT NULL,
  email VARCHAR(150) NOT NULL,
  document_type_id INT NOT NULL,
  document_number VARCHAR(30) NOT NULL,
  token_hash CHAR(64) NOT NULL,
  status VARCHAR(12) COLLATE utf8mb4_bin NOT NULL DEFAULT 'PENDIENTE',
  sent_at DATETIME NOT NULL,
  expires_at DATETIME NOT NULL,
  accepted_at DATETIME NULL,
  created_by VARCHAR(80) NULL,
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  open_enrollment_group_id INT
    GENERATED ALWAYS AS (case when status in ('PENDIENTE','INCOMPLETA') then economic_group_id end) STORED,
  PRIMARY KEY (enrollment_id),
  CONSTRAINT uq_enrollment__open_by_document UNIQUE (document_type_id, document_number, open_enrollment_group_id),
  CONSTRAINT uq_enrollment__token_hash UNIQUE (token_hash),
  CONSTRAINT ck_enrollment__status CHECK (status in ('PENDIENTE','COMPLETADA','INCOMPLETA','EXPIRADA','REVOCADA')),
  CONSTRAINT ck_enrollment__enrollment_type CHECK (enrollment_type in ('USUARIO','ADMINISTRADOR')),
  CONSTRAINT fk_enrollment__company FOREIGN KEY (company_id, economic_group_id)
    REFERENCES belpc_company (company_id, economic_group_id),
  CONSTRAINT fk_enrollment__economic_group FOREIGN KEY (economic_group_id)
    REFERENCES belpc_economic_group (economic_group_id),
  CONSTRAINT fk_enrollment__person_channel FOREIGN KEY (person_id)
    REFERENCES belpc_person_channel (person_id),
  CONSTRAINT fk_enrollment__role FOREIGN KEY (assigned_role_id, economic_group_id)
    REFERENCES belpc_role (role_id, economic_group_id),
  CONSTRAINT fk_enrollment__document_type FOREIGN KEY (document_type_id)
    REFERENCES belpc_document_type (document_type_id),
  CONSTRAINT fk_enrollment__display_type FOREIGN KEY (display_type_id)
    REFERENCES belpc_display_type (display_type_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
  COMMENT='Personas y accesos · Invitaciones (enrollment) a usuarios y administradores. Solo guarda el hash del token.';
CREATE INDEX ix_enrollment__economic_group_id_status ON belpc_enrollment (economic_group_id, status);

-- Relación persona – grupo económico con su rol. Es el usuario operativo.
CREATE TABLE belpc_group_user (
  group_user_id INT NOT NULL AUTO_INCREMENT,
  economic_group_id INT NOT NULL,
  company_id INT NULL,
  person_id VARCHAR(64) NOT NULL COMMENT 'Id de la persona (_id de ForgeRock). Relaciona con BackOffice.',
  role_id INT NOT NULL,
  display_type_id INT NULL,
  enrollment_id INT NULL,
  status VARCHAR(10) COLLATE utf8mb4_bin NOT NULL DEFAULT 'ACTIVO',
  version INT NOT NULL DEFAULT 1,
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  created_by VARCHAR(80) NULL,
  updated_by VARCHAR(80) NULL,
  PRIMARY KEY (group_user_id),
  CONSTRAINT uq_group_user__person_id_company_id UNIQUE (person_id, company_id),
  CONSTRAINT uq_group_user__group_user_id_economic_group_id UNIQUE (group_user_id, economic_group_id),
  CONSTRAINT uq_group_user__group_user_id_economic_group_id_person_id UNIQUE (group_user_id, economic_group_id, person_id),
  CONSTRAINT ck_group_user__status CHECK (status in ('ACTIVO','INACTIVO')),
  CONSTRAINT fk_group_user__enrollment FOREIGN KEY (enrollment_id)
    REFERENCES belpc_enrollment (enrollment_id),
  CONSTRAINT fk_group_user__company FOREIGN KEY (company_id, economic_group_id)
    REFERENCES belpc_company (company_id, economic_group_id),
  CONSTRAINT fk_group_user__economic_group FOREIGN KEY (economic_group_id)
    REFERENCES belpc_economic_group (economic_group_id),
  CONSTRAINT fk_group_user__person_channel FOREIGN KEY (person_id)
    REFERENCES belpc_person_channel (person_id),
  CONSTRAINT fk_group_user__role FOREIGN KEY (role_id, economic_group_id)
    REFERENCES belpc_role (role_id, economic_group_id),
  CONSTRAINT fk_group_user__display_type FOREIGN KEY (display_type_id)
    REFERENCES belpc_display_type (display_type_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
  COMMENT='Personas y accesos · Relación persona – grupo económico con su rol. Es el usuario operativo.';
CREATE INDEX ix_group_user__economic_group_id_status ON belpc_group_user (economic_group_id, status);

-- Administradores del grupo (máximo 2 activos, posiciones 1 y 2).
CREATE TABLE belpc_group_admin (
  group_admin_id INT NOT NULL AUTO_INCREMENT,
  economic_group_id INT NOT NULL,
  person_id VARCHAR(64) NOT NULL COMMENT 'Id de la persona (_id de ForgeRock). Relaciona con BackOffice.',
  admin_position TINYINT NOT NULL,
  group_user_id INT NOT NULL,
  status VARCHAR(10) COLLATE utf8mb4_bin NOT NULL DEFAULT 'ACTIVO',
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  created_by VARCHAR(80) NULL,
  active_admin_position TINYINT
    GENERATED ALWAYS AS (case when status = 'ACTIVO' then admin_position end) STORED,
  PRIMARY KEY (group_admin_id),
  CONSTRAINT uq_group_admin__economic_group_id_person_id UNIQUE (economic_group_id, person_id),
  CONSTRAINT uq_group_admin__economic_group_id_active_admin_position UNIQUE (economic_group_id, active_admin_position),
  CONSTRAINT uq_group_admin__group_user_id UNIQUE (group_user_id),
  CONSTRAINT ck_group_admin__status CHECK (status in ('ACTIVO','INACTIVO')),
  CONSTRAINT ck_group_admin__admin_position CHECK (admin_position in (1,2)),
  CONSTRAINT fk_group_admin__economic_group FOREIGN KEY (economic_group_id)
    REFERENCES belpc_economic_group (economic_group_id),
  CONSTRAINT fk_group_admin__person_channel FOREIGN KEY (person_id)
    REFERENCES belpc_person_channel (person_id),
  CONSTRAINT fk_group_admin__group_user FOREIGN KEY (group_user_id, economic_group_id, person_id)
    REFERENCES belpc_group_user (group_user_id, economic_group_id, person_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
  COMMENT='Personas y accesos · Administradores del grupo (máximo 2 activos, posiciones 1 y 2).';

-- Correo de negocio verificado por OTP de cada usuario del grupo.
CREATE TABLE belpc_notification_contact (
  notification_contact_id INT NOT NULL AUTO_INCREMENT,
  group_user_id INT NOT NULL,
  email VARCHAR(150) NOT NULL,
  otp_verified_at DATETIME NOT NULL,
  status VARCHAR(10) COLLATE utf8mb4_bin NOT NULL DEFAULT 'ACTIVO',
  PRIMARY KEY (notification_contact_id),
  CONSTRAINT uq_notification_contact__group_user_id UNIQUE (group_user_id),
  CONSTRAINT ck_notification_contact__status CHECK (status in ('ACTIVO','INACTIVO')),
  CONSTRAINT fk_notification_contact__group_user FOREIGN KEY (group_user_id)
    REFERENCES belpc_group_user (group_user_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
  COMMENT='Personas y accesos · Correo de negocio verificado por OTP de cada usuario del grupo.';

-- =====================================================================
-- Roles y permisos
-- =====================================================================
-- Permisos del rol; la base exige que estén dentro de su perfil.
CREATE TABLE belpc_role_permission (
  role_permission_id INT NOT NULL AUTO_INCREMENT,
  permission_id INT NOT NULL,
  role_id INT NOT NULL,
  profile_id INT NOT NULL,
  PRIMARY KEY (role_permission_id),
  CONSTRAINT uq_role_permission__role_id_permission_id UNIQUE (role_id, permission_id),
  CONSTRAINT fk_role_permission__profile_permission FOREIGN KEY (profile_id, permission_id)
    REFERENCES belpc_profile_permission (profile_id, permission_id),
  CONSTRAINT fk_role_permission__permission FOREIGN KEY (permission_id)
    REFERENCES belpc_permission (permission_id),
  CONSTRAINT fk_role_permission__role__role FOREIGN KEY (role_id)
    REFERENCES belpc_role (role_id),
  CONSTRAINT fk_role_permission__role__role_profile FOREIGN KEY (role_id, profile_id)
    REFERENCES belpc_role (role_id, profile_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
  COMMENT='Roles y permisos · Permisos del rol; la base exige que estén dentro de su perfil.';

-- Grupos de productos que ve y opera cada rol.
CREATE TABLE belpc_role_product_group (
  role_product_group_id INT NOT NULL AUTO_INCREMENT,
  role_id INT NOT NULL,
  product_group_id INT NOT NULL,
  economic_group_id INT NOT NULL,
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  created_by VARCHAR(80) NULL,
  PRIMARY KEY (role_product_group_id),
  CONSTRAINT uq_role_product_group__role_id_product_group_id UNIQUE (role_id, product_group_id),
  CONSTRAINT fk_role_product_group__product_group FOREIGN KEY (product_group_id, economic_group_id)
    REFERENCES belpc_product_group (product_group_id, economic_group_id),
  CONSTRAINT fk_role_product_group__role FOREIGN KEY (role_id, economic_group_id)
    REFERENCES belpc_role (role_id, economic_group_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
  COMMENT='Roles y permisos · Grupos de productos que ve y opera cada rol.';

-- =====================================================================
-- Flujos de aprobación
-- =====================================================================
-- Flujo de aprobación de transacciones del grupo.
CREATE TABLE belpc_approval_flow (
  approval_flow_id INT NOT NULL AUTO_INCREMENT,
  economic_group_id INT NOT NULL,
  code VARCHAR(30) NOT NULL,
  name VARCHAR(150) NOT NULL,
  version INT NOT NULL DEFAULT 1,
  status VARCHAR(10) COLLATE utf8mb4_bin NOT NULL DEFAULT 'ACTIVO',
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  created_by VARCHAR(80) NULL,
  PRIMARY KEY (approval_flow_id),
  CONSTRAINT uq_approval_flow__economic_group_id_code UNIQUE (economic_group_id, code),
  CONSTRAINT uq_approval_flow__approval_flow_id_economic_group_id UNIQUE (approval_flow_id, economic_group_id),
  CONSTRAINT ck_approval_flow__status CHECK (status in ('ACTIVO','INACTIVO')),
  CONSTRAINT fk_approval_flow__economic_group FOREIGN KEY (economic_group_id)
    REFERENCES belpc_economic_group (economic_group_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
  COMMENT='Flujos de aprobación · Flujo de aprobación de transacciones del grupo.';

-- Reglas por tipo de transacción, moneda y rango de monto (desde inclusivo, hasta exclusivo).
CREATE TABLE belpc_approval_rule (
  approval_rule_id INT NOT NULL AUTO_INCREMENT,
  approval_flow_id INT NOT NULL,
  economic_group_id INT NOT NULL,
  transaction_type_id INT NOT NULL,
  currency_id INT NOT NULL,
  amount_from DECIMAL(18,2) NOT NULL,
  amount_to DECIMAL(18,2) NOT NULL,
  PRIMARY KEY (approval_rule_id),
  CONSTRAINT uq_approval_rule__approval_rule_id_approval_flow_id UNIQUE (approval_rule_id, approval_flow_id),
  CONSTRAINT uq_approval_rule__approval_rule_id_economic_group_id UNIQUE (approval_rule_id, economic_group_id),
  CONSTRAINT ck_approval_rule__amount_from_amount_to CHECK (amount_from >= 0 and amount_to > amount_from),
  CONSTRAINT fk_approval_rule__approval_flow FOREIGN KEY (approval_flow_id, economic_group_id)
    REFERENCES belpc_approval_flow (approval_flow_id, economic_group_id),
  CONSTRAINT fk_approval_rule__currency FOREIGN KEY (currency_id)
    REFERENCES belpc_currency (currency_id),
  CONSTRAINT fk_approval_rule__transaction_type FOREIGN KEY (transaction_type_id)
    REFERENCES belpc_transaction_type (transaction_type_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
  COMMENT='Flujos de aprobación · Reglas por tipo de transacción, moneda y rango de monto (desde inclusivo, hasta exclusivo).';

-- Niveles ordenados de cada regla y cuántas aprobaciones requiere cada nivel.
CREATE TABLE belpc_approval_level (
  approval_level_id INT NOT NULL AUTO_INCREMENT,
  approval_rule_id INT NOT NULL,
  economic_group_id INT NOT NULL,
  level_order INT NOT NULL,
  required_approvals INT NOT NULL,
  PRIMARY KEY (approval_level_id),
  CONSTRAINT uq_approval_level__approval_level_id_economic_group_id UNIQUE (approval_level_id, economic_group_id),
  CONSTRAINT uq_approval_level__approval_rule_id_level_order UNIQUE (approval_rule_id, level_order),
  CONSTRAINT uq_approval_level__approval_level_id_approval_rule_id UNIQUE (approval_level_id, approval_rule_id),
  CONSTRAINT ck_approval_level__required_approvals CHECK (required_approvals > 0),
  CONSTRAINT fk_approval_level__approval_rule FOREIGN KEY (approval_rule_id, economic_group_id)
    REFERENCES belpc_approval_rule (approval_rule_id, economic_group_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
  COMMENT='Flujos de aprobación · Niveles ordenados de cada regla y cuántas aprobaciones requiere cada nivel.';

-- Usuarios del grupo que aprueban en cada nivel.
CREATE TABLE belpc_approval_approver (
  approval_approver_id INT NOT NULL AUTO_INCREMENT,
  approval_level_id INT NOT NULL,
  group_user_id INT NOT NULL,
  economic_group_id INT NOT NULL,
  PRIMARY KEY (approval_approver_id),
  CONSTRAINT uq_approval_approver__approval_level_id_group_user_id UNIQUE (approval_level_id, group_user_id),
  CONSTRAINT fk_approval_approver__approval_level FOREIGN KEY (approval_level_id, economic_group_id)
    REFERENCES belpc_approval_level (approval_level_id, economic_group_id),
  CONSTRAINT fk_approval_approver__group_user FOREIGN KEY (group_user_id, economic_group_id)
    REFERENCES belpc_group_user (group_user_id, economic_group_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
  COMMENT='Flujos de aprobación · Usuarios del grupo que aprueban en cada nivel.';

-- Productos a los que aplica el flujo.
CREATE TABLE belpc_approval_flow_product (
  approval_flow_product_id INT NOT NULL AUTO_INCREMENT,
  approval_flow_id INT NOT NULL,
  linked_product_id INT NOT NULL,
  economic_group_id INT NOT NULL,
  PRIMARY KEY (approval_flow_product_id),
  CONSTRAINT uq_approval_flow_product__approval_flow_id_linked_product_id UNIQUE (approval_flow_id, linked_product_id),
  CONSTRAINT fk_approval_flow_product__approval_flow FOREIGN KEY (approval_flow_id, economic_group_id)
    REFERENCES belpc_approval_flow (approval_flow_id, economic_group_id),
  CONSTRAINT fk_approval_flow_product__linked_product FOREIGN KEY (linked_product_id, economic_group_id)
    REFERENCES belpc_linked_product (linked_product_id, economic_group_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
  COMMENT='Flujos de aprobación · Productos a los que aplica el flujo.';

-- =====================================================================
-- Aprobaciones administrativas
-- =====================================================================
-- Solicitud de cambio administrativo pendiente de aprobación (control dual).
CREATE TABLE belpc_admin_approval (
  admin_approval_id INT NOT NULL AUTO_INCREMENT,
  economic_group_id INT NOT NULL,
  admin_change_type_id INT NOT NULL,
  entity VARCHAR(50) NOT NULL,
  record_id VARCHAR(50) NOT NULL,
  action VARCHAR(20) NOT NULL,
  previous_data JSON NULL,
  proposed_data JSON NULL,
  record_version INT NOT NULL,
  required_approvals INT NOT NULL DEFAULT 1,
  status VARCHAR(10) COLLATE utf8mb4_bin NOT NULL DEFAULT 'PENDIENTE',
  requested_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  expires_at DATETIME NOT NULL,
  resolved_at DATETIME(3) NULL,
  requested_by_person_id VARCHAR(64) NOT NULL COMMENT 'person_id de quien solicita',
  failure_reason VARCHAR(255) NULL,
  correlation_id VARCHAR(64) NULL,
  PRIMARY KEY (admin_approval_id),
  CONSTRAINT uq_admin_approval__admin_approval_id_economic_group_id UNIQUE (admin_approval_id, economic_group_id),
  CONSTRAINT ck_admin_approval__status CHECK (status in ('PENDIENTE','APROBADA','APLICADA','RECHAZADA','EXPIRADA','CANCELADA','FALLIDA')),
  CONSTRAINT ck_admin_approval__required_approvals CHECK (required_approvals > 0),
  CONSTRAINT ck_admin_approval__status_resolved_at CHECK (status = 'PENDIENTE' and resolved_at is null or status <> 'PENDIENTE' and resolved_at is not null),
  CONSTRAINT fk_admin_approval__economic_group FOREIGN KEY (economic_group_id)
    REFERENCES belpc_economic_group (economic_group_id),
  CONSTRAINT fk_admin_approval__admin_change_type FOREIGN KEY (admin_change_type_id)
    REFERENCES belpc_admin_change_type (admin_change_type_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
  COMMENT='Aprobaciones administrativas · Solicitud de cambio administrativo pendiente de aprobación (control dual).';

-- Decisión de cada administrador: quién, cuándo y por qué (evidencia SIB).
CREATE TABLE belpc_admin_approval_decision (
  admin_approval_decision_id INT NOT NULL AUTO_INCREMENT,
  admin_approval_id INT NOT NULL,
  economic_group_id INT NOT NULL,
  person_id VARCHAR(64) NOT NULL COMMENT 'Id de la persona (_id de ForgeRock). Relaciona con BackOffice.',
  decision VARCHAR(10) COLLATE utf8mb4_bin NOT NULL,
  comment VARCHAR(400) NULL,
  decided_at DATETIME(3) NOT NULL,
  PRIMARY KEY (admin_approval_decision_id),
  CONSTRAINT uq_admin_approval_decision__admin_approval_id_person_id UNIQUE (admin_approval_id, person_id),
  CONSTRAINT ck_admin_approval_decision__decision CHECK (decision in ('APROBADA','RECHAZADA')),
  CONSTRAINT fk_admin_approval_decision__group_admin FOREIGN KEY (economic_group_id, person_id)
    REFERENCES belpc_group_admin (economic_group_id, person_id),
  CONSTRAINT fk_admin_approval_decision__admin_approval FOREIGN KEY (admin_approval_id, economic_group_id)
    REFERENCES belpc_admin_approval (admin_approval_id, economic_group_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
  COMMENT='Aprobaciones administrativas · Decisión de cada administrador: quién, cuándo y por qué (evidencia SIB).';

-- =====================================================================
-- Transacciones en espera (R2)
-- =====================================================================
-- Transacción monetaria esperando aprobación según su flujo.
CREATE TABLE belpc_pending_transaction (
  pending_transaction_id INT NOT NULL AUTO_INCREMENT,
  economic_group_id INT NOT NULL,
  approval_flow_id INT NOT NULL,
  approval_flow_version INT NOT NULL,
  approval_rule_id INT NOT NULL,
  requested_by_group_user_id INT NOT NULL,
  amount DECIMAL(18,2) NOT NULL,
  currency_id INT NOT NULL,
  source_account_masked VARCHAR(40) NOT NULL,
  destination_account_masked VARCHAR(40) NOT NULL,
  status VARCHAR(15) COLLATE utf8mb4_bin NOT NULL DEFAULT 'PENDIENTE',
  current_level INT NULL,
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  expires_at DATETIME NOT NULL,
  core_reference VARCHAR(40) NULL,
  version INT NOT NULL DEFAULT 1,
  PRIMARY KEY (pending_transaction_id),
  CONSTRAINT uq_pending_transaction__pending_transaction_id_approval_rule_id UNIQUE (pending_transaction_id, approval_rule_id),
  CONSTRAINT ck_pending_transaction__status CHECK (status in ('PENDIENTE','EN_APROBACION','APROBADA','EJECUTADA','RECHAZADA','EXPIRADA','CANCELADA','FALLIDA')),
  CONSTRAINT ck_pending_transaction__amount CHECK (amount > 0),
  CONSTRAINT fk_pending_transaction__group_user FOREIGN KEY (requested_by_group_user_id, economic_group_id)
    REFERENCES belpc_group_user (group_user_id, economic_group_id),
  CONSTRAINT fk_pending_transaction__approval_flow FOREIGN KEY (approval_flow_id, economic_group_id)
    REFERENCES belpc_approval_flow (approval_flow_id, economic_group_id),
  CONSTRAINT fk_pending_transaction__economic_group FOREIGN KEY (economic_group_id)
    REFERENCES belpc_economic_group (economic_group_id),
  CONSTRAINT fk_pending_transaction__currency FOREIGN KEY (currency_id)
    REFERENCES belpc_currency (currency_id),
  CONSTRAINT fk_pending_transaction__approval_rule FOREIGN KEY (approval_rule_id, approval_flow_id)
    REFERENCES belpc_approval_rule (approval_rule_id, approval_flow_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
  COMMENT='Transacciones en espera (R2) · Transacción monetaria esperando aprobación según su flujo.';

-- Decisión de cada aprobador sobre una transacción en espera.
CREATE TABLE belpc_pending_transaction_decision (
  pending_transaction_decision_id INT NOT NULL AUTO_INCREMENT,
  pending_transaction_id INT NOT NULL,
  approval_rule_id INT NOT NULL,
  approval_level_id INT NOT NULL,
  group_user_id INT NOT NULL,
  decision VARCHAR(10) COLLATE utf8mb4_bin NOT NULL,
  comment VARCHAR(400) NULL,
  decided_at DATETIME NOT NULL,
  PRIMARY KEY (pending_transaction_decision_id),
  CONSTRAINT uq_pending_transaction_decision__pending_transaction_group_user UNIQUE (pending_transaction_id, group_user_id),
  CONSTRAINT ck_pending_transaction_decision__decision CHECK (decision in ('APROBADA','RECHAZADA')),
  CONSTRAINT fk_pending_transaction_decision__approval_approver FOREIGN KEY (approval_level_id, group_user_id)
    REFERENCES belpc_approval_approver (approval_level_id, group_user_id),
  CONSTRAINT fk_pending_transaction_decision__approval_level FOREIGN KEY (approval_level_id, approval_rule_id)
    REFERENCES belpc_approval_level (approval_level_id, approval_rule_id),
  CONSTRAINT fk_pending_transaction_decision__pending_transaction FOREIGN KEY (pending_transaction_id, approval_rule_id)
    REFERENCES belpc_pending_transaction (pending_transaction_id, approval_rule_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
  COMMENT='Transacciones en espera (R2) · Decisión de cada aprobador sobre una transacción en espera.';

-- =====================================================================
-- Datos iniciales
-- =====================================================================
-- 4.2 Perfil del rol de sistema del administrador.
INSERT INTO belpc_profile (code, name, description, created_by)
VALUES ('ADMINISTRADOR', 'Administrador', 'Perfil del rol de sistema del administrador del grupo', 'liquibase');
-- [AJUSTAR] Cargar en belpc_profile_permission solo los permisos de la columna
-- ADMINISTRADOR de la matriz del taller, cuando se cargue belpc_permission.

-- =====================================================================
-- 4.1 [PENDIENTE D1 · NO EJECUTAR] Un rol por persona y grupo económico.
-- Descomentar solo si se decide que la relación es por grupo económico.
-- =====================================================================
-- ALTER TABLE belpc_group_user
--   DROP INDEX uq_group_user__person_id_company_id,
--   ADD CONSTRAINT uq_group_user__person_id_economic_group_id UNIQUE (person_id, economic_group_id);
