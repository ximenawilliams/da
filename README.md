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


* [ ] **Бэкенд (управление отменой):** Разработать логический процесс отмены в базе данных («CANCELLATION») и HTTP-клиент с повторными попытками для использования REST API ForgeRock IDM.

* [ ] **QA:** Проверить тестовые случаи для пользователей с несколькими учетными записями (одна учетная запись, несколько компаний), убедившись, что внешние ключи правильно указывают на один и тот же `id_persona`.

