🟢 HU-01 · Consultar catálogo · GET /v1/catalogos/{tipo}
Como pantalla de BELC, quiero los valores de un catálogo para llenar las listas desplegables.

Catálogos: tipos-documento, monedas, tipos-transaccion, perfiles, permisos, tipos-visualizacion, tipos-producto.

Criterios:

Devuelve código y nombre, solo de los valores con estado ACTIVO. permisos devuelve además su categoría.
Un {tipo} que no está en la lista responde 404.
Cada catálogo sale de su tabla en BackOffice; no hay llamadas a COBIS ni a ForgeRock.

Antes de empezar: confirmar si BELP ya tiene un servicio de catálogos, y agregar Estado a Tipo_transaccion.

Ejemplo de respuesta (GET /v1/catalogos/permisos):

json
{
  "tipo": "permisos",
  "valores": [
    { "codigo": "CREAR_ROLES", "nombre": "Creación de roles", "categoria": "ADMINISTRATIVO" },
    { "codigo": "DESCARGAR_ESTADO_CUENTA", "nombre": "Descargar estado de cuenta", "categoria": "MONETARIO" }
  ]
}



🟢 HU-02 · Crear grupo económico

Microservicio: ms-grupos-economicos · Endpoint: POST /v1/grupos-economicos

Como empleado de Servicio de Canales, quiero registrar un grupo económico, para iniciar la afiliación de sus empresas a BEL Comercial.

Componentes
Componente	Qué hace en esta historia
BackOffice web (pantalla del banco)	Formulario con nombre y tipo de administración
API Gateway	Valida el token del empleado y enruta la petición al microservicio
ms-grupos-economicos	Valida los datos, guarda el grupo y registra el evento
Base BELC (Aurora MySQL)	Tablas Grupo_economico, BITACORA_EVENTO y TIPO_EVENTO
Librería de bitácora (HU-13)	Escribe el evento en la misma transacción que el grupo
ForgeRock, COBIS	No intervienen en esta historia
Modelo de datos

Grupo_economico (inserta una fila)

Campo	Valor
id_grupo_economico	Lo genera la base (PK)
Nombre	Viene del formulario; obligatorio, máximo 100 caracteres
Tipo_administracion	I (Individual) o D (Dual), con CHECK
Fecha_vinculacion	Fecha del servidor

BITACORA_EVENTO (inserta una fila)

Campo	Valor
id_tipo_evento	→ TIPO_EVENTO con código GRUPO_ECONOMICO_CREADO y categoría BACKOFFICE
Resultado	Exito o Fallo
Canal	BO
id_grupo_economico	El id del grupo recién creado
Usuario_backoffice	El usuario del empleado
id_correlacion	El correlationId de la petición
Entidad, id_registro	Grupo_economico y su id
Valor_nuevo	El JSON con nombre y tipo
Fecha_evento_utc	La fecha en UTC
Contrato

Petición

json
{ "nombre": "Grupo Pacífico", "tipoAdministracion": "D" }

Respuesta 201

json
{ "idGrupoEconomico": 20, "nombre": "Grupo Pacífico", "tipoAdministracion": "D", "fechaVinculacion": "2026-10-27" }

Errores: 400 datos inválidos · 401 sin sesión · 403 sin permiso · 409 ya existe un grupo con ese nombre.

Flujo
El empleado envía el formulario.
El API Gateway valida su token.
El microservicio valida el permiso, que el nombre no esté vacío y que el tipo sea I o D.
En una sola transacción: inserta en Grupo_economico y en BITACORA_EVENTO.
Responde 201 con el id del grupo.
Criterios de aceptación
Guarda el nombre y el tipo de administración (I o D). Cualquier otro tipo responde 400.
No permite dos grupos con el mismo nombre, aunque cambien mayúsculas o espacios (“Grupo Pacífico” y “grupo pacífico “ son el mismo). Responde 409.
Devuelve el idGrupoEconomico generado.
Si falla la escritura en la bitácora, el grupo tampoco se guarda.
Un empleado sin permiso recibe 403, y el intento queda en la bitácora con resultado Fallo.
Pendientes antes de empezar
Agregar un UNIQUE en Grupo_economico.Nombre. En tu diagrama no aparece, y sin él el criterio 2 depende solo del código.
¿Cómo se autentica el empleado del banco? No es un cliente BELC, así que no entra por el journey de ForgeRock. Probablemente usa el login que ya tiene el BackOffice del banco, con mdl02_employee. Confirmarlo con el líder técnico.
El nombre del permiso del empleado (por ejemplo, CREAR_GRUPO_ECONOMICO) y dónde se guarda: en el modelo de empleados existente o en uno nuevo.
Agregar GRUPO_ECONOMICO_CREADO a TIPO_EVENTO en tus inserts.

//////

🟢 HU-01 · Consultar catálogo · GET /v1/catalogos/{tipo}
Como pantalla de BELC, quiero los valores de un catálogo para llenar las listas desplegables.

Catálogos: tipos-documento, monedas, tipos-transaccion, perfiles, permisos, tipos-visualizacion, tipos-producto.

Criterios:

Devuelve código y nombre, solo de los valores con estado ACTIVO. permisos devuelve además su categoría.
Un {tipo} que no está en la lista responde 404.
Cada catálogo sale de su tabla en BackOffice; no hay llamadas a COBIS ni a ForgeRock.

Antes de empezar: confirmar si BELP ya tiene un servicio de catálogos, y agregar Estado a Tipo_transaccion.

Ejemplo de respuesta (GET /v1/catalogos/permisos):

json
{
  "tipo": "permisos",
  "valores": [
    { "codigo": "CREAR_ROLES", "nombre": "Creación de roles", "categoria": "ADMINISTRATIVO" },
    { "codigo": "DESCARGAR_ESTADO_CUENTA", "nombre": "Descargar estado de cuenta", "categoria": "MONETARIO" }
  ]
}



////

HU 1 · Registrar a la persona en BELC (belc_persona_canal)

Como backend BELC, quiero un registro local mínimo de cada persona, ligado a su identidad de ForgeRock, para relacionarla con sus empresas, roles y aprobaciones sin duplicar sus datos personales.

Campos y para qué sirven

id_persona: PK interna que genera BackOffice. Todas las tablas de BELC se relacionan con la persona por esta llave.
Identity_id: el _id de ForgeRock, UNIQUE. Es la conexión con la identidad.
Estado_canal: bloquea las operaciones BELC desde BackOffice, aunque el token de sesión siga vigente.
Nombre_mostrar: el nombre que se muestra en pantallas, bandejas y bitácora. Es una copia de ForgeRock y no es fuente de verdad.
Terms_version y Terms_accepted_at: evidencia de qué términos aceptó la persona y cuándo.
No se guardan el documento, el correo de seguridad ni el teléfono. Se consultan en ForgeRock cuando hacen falta.

Cuándo se escribe

Al registrarse (nodo 200): si el Identity_id no existe, se crea la fila con un id_persona nuevo. Si ya existe, se reutiliza su id_persona.
En cada login: se actualiza Nombre_mostrar solo si cambió en ForgeRock. No hay copia periódica.
En la baja: Estado_canal = 'INACTIVO' y después se propaga a IDM belc.enabled = false.
Conciliación diaria: se reintentan las bajas que no llegaron a IDM.

Criterios de aceptación

Un Identity_id queda asociado a un solo id_persona.
Una persona en varias empresas tiene una sola fila.
Un reintento con el mismo enrollment_id no duplica la fila.
Si el nombre cambió, el login lo actualiza; si no cambió, no se escribe nada.
No se escribe en mdl01_client ni se llama a COBIS.

Dependencia: ForgeRock tiene que confirmar si el nombre viaja en el token de sesión o hay que leerlo de /userinfo.

HU 2 · Resolver la persona en cada operación

Como backend BELC, quiero obtener el id_persona a partir del identity_id del token de sesión y validar su estado, para autorizar cada operación con las llaves internas de BackOffice.

Flujo

Se valida el token de sesión (audiencia y belc_enabled).
Se busca el id_persona en belc_persona_canal usando Identity_id.
Se valida que Estado_canal = 'ACTIVO'.
Con el id_persona, se consultan la relación con la empresa seleccionada y sus permisos.

Criterios de aceptación

Un identity_id sin fila en belc_persona_canal se rechaza.
Si Estado_canal = 'INACTIVO', la operación se rechaza aunque el token tenga belc_enabled = true.
Sin una relación activa con la empresa seleccionada, la operación se rechaza.
Los rechazos quedan registrados en BITACORA_EVENTO.

Depende de: HU 1.


/////////////////
Probablemente los dos tienen razón, porque están hablando de dos "tokens" distintos. Es una confusión muy común con ForgeRock.
Los dos tokens


Soft token (segundo factor)
Token de sesión (SSO)
Qué es
La app en el celular que genera el código o aprueba el ingreso
El access token que ForgeRock emite al iniciar sesión
Dónde vive
En el celular, registrado a la identidad (authDeviceProfiles = TOKEN)
En la sesión del navegador o de la app; dura minutos u horas
Quién lo menciona
La PO: "no se pueden tener 2 tokens en un celular"
El proveedor: "con SSO sí se puede"

Por qué la PO tiene razón (en el modelo anterior). Cuando había un usuario por contrato, un cliente con 7 contratos tenía 7 usuarios, y cada uno necesitaba su propio soft token. Si la app del banco solo permite registrar un token por celular, esa persona no podía tener los 7 en el mismo teléfono. Ese era un problema real.
Por qué el proveedor tiene razón (en el modelo nuevo). Con la identidad única ya no hacen falta dos tokens:
Un solo soft token por persona. El documento de arquitectura dice que el soft token es de la identidad y es el mismo en todos los portales. Ana, la contadora, usa el mismo token en el celular para BELP, para BELC y para sus tres empresas.
Un solo token de sesión. Con SSO, Ana inicia sesión una vez y ForgeRock le emite un solo access token, válido para BELP y BELC (belp_enabled y belc_enabled). Al pasar de un canal a otro no vuelve a autenticarse.
Las empresas no generan tokens. La empresa se elige después del login y se valida en BackOffice. Por eso tener más empresas no agrega tokens.
En otras palabras, la restricción de la PO sigue siendo cierta (un token por celular), pero con el nuevo modelo ya no es un problema, porque cada persona necesita un solo token.
El único caso en que vuelve a aparecer el problema: cuando una misma persona tiene dos identidades en ForgeRock, por ejemplo por un registro duplicado del modelo anterior. Ahí sí tendría dos soft tokens. Por eso el documento de arquitectura exige fusionar las identidades duplicadas antes de invitar a alguien a BELC, y buscar siempre por cédula antes de crear una identidad nueva.
Para cerrar la discusión en la mesa, pregunta tres cosas:
A la PO: ¿te refieres al soft token del celular o a la sesión?
Al proveedor: ¿el soft token de BELC es el mismo que ya usa la persona en BELP, o se registra uno nuevo?
A Seguridad: ¿el segundo factor del login de BELC será el soft token o un OTP? El documento de arquitectura lo deja como decisión pendiente de Seguridad.
Ojo: no sé si la app del banco permite técnicamente registrar más de un token en el mismo celular. Eso depende de cómo esté construida, y conviene que lo confirme el proveedor. Pero con la identidad única no debería hacer falta.
Una frase para la mesa: "La restricción de un token por celular venía del modelo de un usuario por contrato. Con la identidad única, cada persona tiene un solo soft token y una sola sesión SSO para BELP y BELC, sin importar con cuántas empresas trabaje."
Exacto, aunque con un matiz importante en los perfiles de ForgeRock.
ForgeRock responde "quién eres y por dónde puedes entrar".
Una identidad por persona, con su contraseña, su soft token y su MFA.
Los perfiles en customProfile son por canal, no por empresa: belp y belc. Solo dicen si la persona tiene habilitado cada canal.
Emite un único token de sesión con identity_id, belp_enabled y belc_enabled.
BackOffice responde "con quién trabajas y qué puedes hacer".
belc_persona_canal: la persona en BELC, con su id_persona.
Usuario_grupo_economico: cada empresa con la que trabaja y su rol.
Roles, permisos, grupos de productos y flujos de aprobación.
Las solicitudes en espera de firmas, hasta que van a COBIS.
COBIS responde "qué hay en las cuentas". Las empresas, los productos, los saldos y la ejecución de las transacciones.
El matiz importa porque es justo donde se equivocó el modelo de CALASIS: ahí los perfiles eran por empresa (Empresa 1, Empresa 2), y eso metía datos de negocio en ForgeRock. En BELC, si mañana Ana empieza a trabajar con una cuarta empresa, ForgeRock no cambia nada: solo se agrega una fila en BackOffice. Esa separación es lo que hace que funcionen al mismo tiempo la afiliación múltiple, el token único y el SSO.
En la mesa lo puedes resumir así: "ForgeRock autentica, BackOffice autoriza, COBIS ejecuta."
////
IDM significa Identity Management (gestión de identidades). Es uno de los tres componentes de la plataforma de ForgeRock, y cada uno tiene un trabajo distinto:
Componente
Qué hace
Analogía
AM (Access Management)
Autentica: los journeys de registro y login, el MFA, el soft token, la sesión y la emisión del token SSO
El guardia de la entrada
IDM (Identity Management)
Administra las identidades: define qué datos tiene una persona, los crea, los modifica y los expone por API
El departamento de recursos humanos
DS (Directory Services)
Guarda físicamente los datos, en un directorio LDAP
El archivo donde están los expedientes

Qué hace IDM en tu proyecto, concretamente:
Define el objeto managed/user, es decir, la ficha de la persona: _id, userName, givenName, sn, mail, telephoneNumber, documento y customProfile. Esa definición vive en managed.json.
Decide cómo se guarda en DS. Eso está en repo.ds.json. Por ejemplo, el OIDM de CALASIS muestra que customProfile se guarda como un atributo LDAP de tipo JSON.
Ofrece una API REST para crear, consultar y modificar identidades. Es la que usará BackOffice con su cuenta de servicio para la baja de BELC (customProfile.belc.enabled = false).
Lo usan los journeys de AM. Cuando el journey de registro crea a una persona nueva o activa BELC, lo hace a través de IDM, con los nodos Create Object y Patch Object.
Puede ejecutar scripts ante eventos, por ejemplo al modificar una identidad. Es la base de la opción con Lambda que quedó en el documento.
Lo que IDM no hace en tu diseño: manejar roles, permisos ni empresas. El OIDM lo dice explícitamente: en el banco, IDM se usa solo para la identidad, y la seguridad basada en roles la maneja otra aplicación, que en BELC es BackOffice.
Para la frase que te di antes: cuando decimos "ForgeRock autentica", el que autentica es AM, y la ficha de la persona que usa para hacerlo la administra IDM.

-- Бизнес-параметры, доступные для редактирования через бэк-офис
-- («Экран настройки параметров» в рамках этапа R1).
-- Если id_grupo_economico равно NULL, используется общее значение; если задано конкретное значение, оно переопределяет общее значение для данной группы.
CREATE TABLE Configuration_Parameter (
  id_parametro INT NOT NULL AUTO_INCREMENT,
  Code VARCHAR(50) NOT NULL,
  Description VARCHAR(200) NOT NULL,
  Data_type VARCHAR(10) NOT NULL,
  Value VARCHAR(100) NOT NULL,
  id_grupo_economico INT NULL,
  Status VARCHAR(10) NOT NULL DEFAULT 'ACTIVE',
  Modified_by VARCHAR(80) NULL,
  Modification_date DATETIME(3) NULL ON UPDATE CURRENT_TIMESTAMP(3),
  Version INT NOT NULL DEFAULT 1,
  -- В MySQL ограничение UNIQUE допускает наличие нескольких значений NULL; для корректной работы используется вычисляемый столбец
  group_key INT GENERATED ALWAYS AS (IFNULL(id_grupo_economico, 0)) STORED,
  CONSTRAINT pk_parametro PRIMARY KEY (id_parametro),
  CONSTRAINT uq_parametro UNIQUE (Code, group_key),
  CONSTRAINT ck_parametro_tipo CHECK (Data_type IN ('INTEGER','DECIMAL','TEXT','BOOLEAN')),
  CONSTRAINT ck_parametro_estado CHECK (Status IN ('ACTIVE','INACTIVE')),
  CONSTRAINT fk_parametro_grupo FOREIGN KEY (id_grupo_economico)
    REFERENCES Economic_Group (id_grupo_economico)
) ENGINE = InnoDB DEFAULT CHARACTER SET = utf8mb4; INSERT INTO Parametro_configuracion (Codigo, Descripcion, Tipo_dato, Valor) VALUES
  ('AFIL_HORAS_EXPIRACION', 'Срок действия приглашения (в часах)', 'INTEGER', '72'),
  ('AFIL_MAX_REENVIOS', 'Максимальное количество повторных отправок приглашения', 'INTEGER', '3'),
  ('APR_ADM_HORAS_EXPIRACION', 'Срок действия административного одобрения (в часах)', 'INTEGER', '48');
-- Предлагаемые начальные значения; в дальнейшем их можно изменить через панель администратора (back office).



///////////////////////
Вот формальное, структурированное предложение, написанное в формате «пользовательской истории», готовое к копированию и вставке в Jira. Оно написано с уровнем технической детализации, необходимым для технического руководителя, а также разработчиков бэкенда и баз данных.

---

### **Название в Jira:**

> **[BELC] Внедрение и управление локальным реестром идентификационных данных (`belc_persona_canal`)**

---

### **Тип инцидента:**

* История/Пользовательская история

### **Эпик/Компонент:**

*Модуль идентификации и аутентификации / BELC BackOffice

---

### **Описание / Пользовательская история:**

> **Как** системный архитектор/технический руководитель BELC,
> **Я хочу** реализовать локальную справочную таблицу `belc_persona_canal`, связанную с ForgeRock `Identity_id`, используя внутренний ключ (`id_persona`),
> **Чтобы** BackOffice мог независимо управлять связями, разрешениями, утверждениями и аудитами, отделяя транзакционные данные от центрального каталога и защищая производительность базы данных без дублирования конфиденциальной информации.

---

### **Критерии приемки:**

1. **Структура таблицы (`belc_persona_canal`):**
* Таблица должна быть создана в Aurora MySQL с внутренним первичным ключом `id_persona` (автоматически увеличивающееся целое число или последовательность).

* Она должна включать столбец `Identity_id` (связанный с `_id` в ForgeRock) с уникальным индексом (`UNIQUE`) для обеспечения двунаправленности.

* Она должна хранить только минимальный набор справочных данных: `Estado_canal` (`PENDIENTE_HABILITACION`, `ACTIVO`, `BAJA`), `Nombre_display` (в качестве визуального кэша) и подтверждение условий использования (`Terms_version`, `Terms_accepted_at`).


* **Ограничение на использование персональных данных:** Категорически запрещено дублировать официальные документы, удостоверяющие личность, личные электронные адреса, номера телефонов или учетные данные в этой таблице; эти данные извлекаются непосредственно из ForgeRock по запросу.

2. **Ссылочная целостность (внешние ключи):**
* Зависимые таблицы в бэк-офисе (такие как `affiliation`, `Usuario_grupo_economico`, `Grupo_economico_administrador` и решения об утверждении) должны указывать на `id_persona` с использованием внешних ключей во избежание появления "осиротевших" записей.

3. **Механизм обновления и синхронизации (версия 1):**
* **Запись (путь - шаг 200/190):** При связывании человека бэкэнд ищет `Identity_id`; Если запись отсутствует, она создается в состоянии `PENDING_ACTIVATION`, которое изменяется на `ACTIVE` после активации канала.

* **Вход в систему:** Бэкенд проверит токен сессии, найдет строку по `Identity_id`, получит внутренний `id_persona` и обновит поле `Nombre_distar` в режиме реального времени, если оно изменилось в ForgeRock.

* **Удаление канала:** Бэкенд локально обновит `Estado_canal = 'BAJA'`, чтобы немедленно заблокировать операции, асинхронно запуская (с повторными попытками) вызов REST API IDM (`customProfile.belc.enabled = false`).

---

### **Технические задачи (подзадачи / технические задачи) для разработчиков:**

* **Администратор базы данных / Бэкенд:** Выполнить DDL-скрипт для создания таблицы `belc_persona_canal` и соответствующих ей уникальных индексов по `Identity_id`.

* * [ ] **Бэкенд (Аутентификация/Безопасность):** Изменить перехватчик/промежуточное ПО проверки JWT-токенов для извлечения `Identity_id`, разрешения локального `id_persona` и выполнения `upset` или легковесного обновления `Display_Name` во время входа в систему.

* [ ] **Бэкенд (Интеграция с Journey):** Реализовать логику в службе членства/связывания для регистрации личности в состоянии ожидания и проверки принятия условий («Terms_version»).


//////////////////////////
Эпик: [BELC] Локальный реестр идентификационных данных (belc_persona_canal)

Цель: Для того, чтобы бэк-офис имел уникальный идентификатор для каждого человека в BELC, связанный с его идентификацией в ForgeRock, без дублирования персональных данных. Это служит основой для множественных связей (один человек в нескольких компаниях и группах).

Ссылки: Документ по архитектуре пути BELC (Приложение C, Контракты; Приложение E, Приемочные тесты) · ddl_relaciones_belc_aurora.sql · ER-диаграмма R1, область 1.

Правило эпика: belc_persona_canal хранит только id_persona, Identity_id, Channel_Status, Display_Name и подтверждение условий. Документ, электронная почта, номер телефона и учетные данные хранятся только в ForgeRock.


История 1 · [BELC] Разрешение таблиц и пользователей в API для belc_persona_canal

Тип: Техническая история (внедрение) Компонент: BELC BackOffice · База данных · Безопасность

Описание

Для обеспечения связи всех таблиц BackOffice с пользователем через внутренний ключ и проверки активности канала каждой операцией BELC, платформа BELC должна создать belc_persona_canal с его внешними ключами и разрешать identity_id → id_persona в каждом запросе.

Критерии приемки
gherkin
Сценарий: Структура таблицы

В Aurora MySQL существует `belc_persona_canal` с `id_persona` в качестве первичного ключа.

`Identity_id` является уникальным и не равен NULL.

`State_canal` принимает только значения `PENDING_ACTIVATION`, `ACTIVE` или `INACTIVE`.


В таблице отсутствуют столбцы для документов, электронной почты, телефонов и учетных данных.

Сценарий: Ссылочная целостность

Столбцы `Affiliation`, `Userio_grupo_economico`, `Grupo_economico_administrador` и `Aprobacion_administrativa_decision` ссылаются на `id_persona` через внешний ключ.

База данных отклоняет связь с несуществующим `id_persona`.

Сценарий: Активное лицо использует API BELC

При наличии действительного токена с `identity_id` у X,

у X есть `State_canal` ACTIVE.

При вызове защищенного API BELC,

запрос продолжается с `id_persona` X в контексте.

Сценарий: Идентификация без записи BELC

Предоставлен действительный токен с `identity_id`, но нет записи в belc_persona_canal

При вызове защищенного API BELC

Возвращается ошибка 403 с форматом сведений о проблеме (RFC 9457)

И строка не создается

Сценарий: Канал неактивен с действительным токеном

Поскольку у X статус канала INACTIVE и действительный токен

При вызове защищенного API BELC

Возвращается ошибка 403

Сценарий: Канал ожидает активации

Поскольку у X статус канала PENDING_ACTIVATION

При вызове защищенного API BELC

Возвращается ошибка 403

Сценарий: Устаревшее отображаемое имя

Поскольку имя X изменилось в ForgeRock

При входе X в систему

Отображаемое имя обновляется текущим именем

Технические задачи

DBA: Изменение в Liquibase belc_persona_canal, его индексы и внешние ключи зависимых таблиц (согласно ddl_relaciones_belc_aurora.sql).

Бэкенд: Промежуточное ПО, которое считывает утверждение identity_id, разрешает id_persona и Estado_canal и запрещает доступ, если ID не существует или неактивен.

Бэкенд: Кэш разрешения с коротким сроком действия и немедленной инвалидацией при изменении Estado_canal (используется в Story 3).

Бэкенд: Обновляет display_name при входе в систему на основе решения зависимости D2.
QA: Предыдущие сценарии, включая человека с отношениями в двух компаниях, который разрешается в один и тот же id_persona.

Вне области видимости:
Создание строки: Это делает только Story 2 (вход в систему никогда не создает строки).
Разрешения для каждой компании: Проверяются в бэкенде с помощью user_group_economic_user. История 2 · [BELC] Добавление пользователя путем привязки приглашения (Путь, Шаги 200 и 190)

Тип: История Компонент: Интеграция BELC BackOffice · ForgeRock

Описание

Я, как пользователь, приглашенный для работы от имени компании, хочу быть связан с этой компанией с помощью своей уникальной идентификации после завершения регистрации, чтобы я мог входить в BELC с моими обычными учетными данными, даже если я работаю с несколькими компаниями.

Контракты
Привязка пользователя (шаг 200, уже определен в Приложении C): получает identity_id, enrollment_id, document, display_name, terms_version, terms_accepted_at. Идемпотентно по enrollment_id.

Подтверждение активации (новое, после шага 190): получает identity_id и enrollment_id. Изменяет статус пользователя на АКТИВНЫЙ. Идемпотентно.

Критерии принятия
Ключевые слова
Сценарий: Первое приглашение нового пользователя

Учитывая, что связь с enrollment_id E1 находится в статусе PENDING и активна

И документ соответствует нормализованной форме приглашения

Когда процесс вызывает функцию «Связать пользователя с identity_id X»

Тогда в одной транзакции:

создается учетная запись belc_persona_canal для X в статусе PENDING_ACTIVATION с ее условиями

связь E1 завершается с id_persona и Acceptance_Date

создается связь в User_economic_group с компанией и ролью E1

создается контактное лицо для уведомления о подтверждении с приглашением по электронной почте

событие регистрируется

Сценарий: Подтверждение активации

Учитывая, что X находится в статусе PENDING_ACTIVATION

Когда процесс


* [ ] **Бэкенд (управление отменой):** Разработать логический процесс отмены в базе данных («CANCELLATION») и HTTP-клиент с повторными попытками для использования REST API ForgeRock IDM.

* [ ] **QA:** Проверить тестовые случаи для пользователей с несколькими учетными записями (одна учетная запись, несколько компаний), убедившись, что внешние ключи правильно указывают на один и тот же `id_persona`.

