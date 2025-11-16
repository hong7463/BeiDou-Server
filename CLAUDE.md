# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

BeiDou-Server is a MapleStory v83 private server implementation based on Cosmic, with modern Java architecture. It's a multi-module project with a Java backend (gms-server) using Spring Boot 3 and a Vue.js frontend (gms-ui).

## Build & Run Commands

### Server (gms-server)

**Prerequisites:**
- Java 21 (OpenJDK recommended)
- MySQL 8+ running on localhost:3306
- Maven 3.x

**Build:**
```bash
mvn clean package
```
This creates `gms-server/target/BeiDou.jar`

**Run:**
```bash
cd gms-server
java -jar target/BeiDou.jar
```

**Run with external config:**
```bash
java -jar target/BeiDou.jar --spring.config.location=file:./application.yml
```

**Test:**
```bash
mvn test
```

**Run single test:**
```bash
mvn test -Dtest=ClassName#methodName
```

### Frontend (gms-ui)

**Prerequisites:**
- Node.js v20.15.0 (LTS)
- Yarn

**Install dependencies:**
```bash
cd gms-ui
yarn install
```

**Development server:**
```bash
yarn dev
```
Runs on http://localhost:8787

**Build for production:**
```bash
yarn build
```

**Type checking:**
```bash
yarn type:check
```

**Lint:**
```bash
yarn lint-staged
```

## Architecture

### Multi-Tier Server Design

BeiDou uses a **hierarchical game server architecture**:

1. **Server (Singleton)**: Top-level coordinator managing all worlds
2. **World**: Independent game world with its own economy, guilds, rankings (port = base_port + world_id * 100)
3. **Channel**: Game instance within a world (port = 7575 + channel_id + world_id * 100)

Each channel runs independently with its own:
- MapManager (lazy-loaded map instances)
- EventScriptManager (event instances)
- PlayerStorage (online players)
- ServicesManager (channel-specific services)

### Networking Layer (Netty)

**Two server types:**
- **LoginServer** (port 8484): Handles authentication and character selection
- **ChannelServer** (dynamic ports): Handles actual gameplay

**Packet processing:**
- `PacketProcessor`: Registry mapping `RecvOpcode` → `PacketHandler`
- 150+ packet handlers for game operations
- Custom MapleStory encryption: `MapleAESOFB` + `MapleCustomEncryption`
- Netty pipeline: `PacketDecoder` → Handler → `PacketEncoder`

### Database Layer

**Technology:**
- MyBatis-Flex 1.8.9 (modern alternative to MyBatis-Plus)
- Druid connection pool
- Flyway for schema migrations (70+ migration files in `src/main/resources/db/migration/`)

**Key patterns:**
- Entities in `org.gms.dao.entity` (annotated with `@Table`)
- Mappers in `org.gms.dao.mapper` (MyBatis-Flex auto-generated)
- Services in `org.gms.service` handle business logic
- Controllers in `org.gms.controller` expose REST APIs

**Database initialization:**
- Auto-creates database on first startup if not exists
- Flyway runs migrations automatically
- Versioned migration files: `V{version}__{description}.sql`

### REST API Layer

**Access:**
- Base URL: http://localhost:8686
- Swagger UI: http://localhost:8686/swagger-ui/index.html
- All endpoints require JWT authentication (except login/register)

**API versioning:**
- Paths use `ApiConstant.LATEST` (currently "v1")
- Format: `/{version}/endpoint`
- To add new API version: update `ApiConstant.LATEST` and preserve old endpoints with explicit version tags

**Key controllers:**
- `AuthController`: JWT-based login/register
- `ServerController`: Server control (start/stop worlds/channels)
- `CharacterController`: Character management
- `ConfigController`: Dynamic game configuration

### Scripting Engine (GraalVM JS)

**Architecture:**
- GraalVM JS 23.0.4 with ScriptEngine API
- Per-client script engine caching for performance
- Language-aware script loading based on `ServiceProperty.language`

**Script locations:**
- Primary: `scripts-{language}/` (e.g., `scripts-zh-CN/`)
- Fallback: `scripts/`

**Script types:**
- `npc/`: NPC conversations
- `quest/`: Quest actions (start, complete, abort)
- `portal/`: Portal scripts
- `reactor/`: Reactor actions
- `item/`: Item usage scripts
- `event/`: Event instances
- `map/`: Map-specific scripts

**Manager classes:**
- `NPCScriptManager`: Handles NPC conversations via `NPCConversationManager`
- `QuestScriptManager`: Quest progression via `QuestActionManager`
- `EventScriptManager`: Event instances via `EventInstanceManager`

### Internationalization

**Supported languages:** zh-CN (Chinese), en-US (English)

**Resource bundles:**
- `src/main/resources/i18n/message_{locale}.properties`: User-facing messages
- `src/main/resources/i18n/log_{locale}.properties`: Server logging
- `src/main/resources/i18n/exception_{locale}.properties`: Exception messages

**Configuration:**
- Set language in `application.yml`: `gms.service.language: zh-CN`
- Affects both scripts and WZ data loading (looks for `wz-{language}/` directories)

**Usage in code:**
```java
I18nUtil.getLogMessage("key.name", arg1, arg2)
I18nUtil.getExceptionMessage("error.key")
```

### Configuration Management

**Three-tier configuration:**

1. **application.yml**: Infrastructure settings (database, ports, JWT secret, Swagger)
2. **game_config table**: Dynamic game parameters with hot-reload capability
   - Hierarchical: type/subtype/code
   - World-specific or server-wide
   - JSON values with type metadata
3. **ServiceProperty**: Runtime settings loaded from application.yml

**Modifying game config at runtime:**
- Via REST API: `POST /v1/config/update`
- Via database: Update `game_config` table
- Some configs support hot-reload without restart (rates, messages)

### Data Loading Strategy

**Initialization sequence (Server.init()):**
1. Register channel dependencies
2. **Parallel WZ data loading** using virtual threads:
   - Skills (`SkillFactory`)
   - Cash items (`CashItemFactory`)
   - Quests (`Quest.loadAllQuests()`)
   - Skillbooks
3. Reset login/merchant states
4. Load coupon rates
5. Initialize worlds (read from `game_config`)
6. Initialize family system (if enabled)
7. Start LoginServer
8. Load event scripts for all channels

**Lazy loading:**
- Maps loaded on first access via `MapFactory`
- Scripts loaded on first invocation
- Item/Monster/NPC data cached by information providers

### Core Package Organization

```
org.gms/
├── client/          # Player state, inventory, skills, buffs
├── config/          # Spring configurations (Security, CORS, I18n)
├── constants/       # Game constants, opcodes
├── controller/      # REST API endpoints
├── dao/             # Entities and MyBatis-Flex mappers
├── exception/       # Custom exception handling
├── manager/         # ServerManager (bootstraps game server)
├── model/           # DTOs and POJOs
├── net/             # Networking layer
│   ├── encryption/  # Packet encryption
│   ├── netty/       # Netty server implementations
│   ├── opcodes/     # SendOpcode, RecvOpcode
│   ├── packet/      # Packet abstractions
│   └── server/      # Server singleton, World, Channel
├── scripting/       # Script management
│   ├── event/       # Event instances
│   └── [type]/      # Script managers by type
├── server/          # Game world logic
│   ├── events/      # GM events (Ola, OxQuiz, Fitness, etc.)
│   ├── expeditions/ # Boss expeditions
│   ├── life/        # Monsters, NPCs (LifeFactory)
│   ├── loot/        # Drop/loot system
│   ├── maps/        # Map instances (MapFactory, MapManager)
│   └── quest/       # Quest system
├── service/         # Business logic (30+ services)
└── util/            # Utility classes
```

### Key Singletons & Factories

**Singletons:**
- `Server`: Central game coordinator (manages worlds)
- `SkillFactory`: Skill data cache
- `ItemInformationProvider`: Item data cache
- `MonsterInformationProvider`: Monster data cache

**Factories:**
- `ItemFactory`: Creates items from database records
- `LifeFactory`: Creates monsters/NPCs from WZ data
- `MapFactory`: Loads map instances
- `CashItemFactory`: Cash shop items

### Concurrency Patterns

- `ReentrantReadWriteLock` for world/login state
- `ReadWriteLock` for merchant operations
- Lock-protected collections for guilds/alliances
- Thread-safe `PlayerStorage` for online player tracking
- Virtual threads (Java 21) for parallel WZ loading

## Development Workflow

### Working with Database Migrations

**Creating a new migration:**
1. Create file: `src/main/resources/db/migration/V{version}__{description}.sql`
2. Version format: `1.0.{next_number}` (e.g., `V1.0.52__add_new_table.sql`)
3. Flyway runs it automatically on next startup

**Note:** Flyway validation is disabled in application.yml for development flexibility.

### Adding New REST Endpoints

1. Create controller in `org.gms.controller`
2. Use `@Tag(name = ApiConstant.LATEST)` for versioning
3. Use `@RequestMapping("/" + ApiConstant.LATEST + "/your-path")`
4. Return `ResultBody<T>` for consistent response format
5. Secure with `@PreAuthorize` if needed

### Modifying Packet Handlers

1. Locate handler in `org.gms.net.server.channel.handlers` or `org.gms.net.server.login.handlers`
2. Packet handlers implement `PacketHandler` interface
3. Register in `PacketProcessor` registry
4. Use `InPacket` for reading, `PacketCreator` or specific factories for responses

### Adding New Scripts

1. Place script in appropriate directory under `scripts-{language}/`
2. Use JavaScript ES5 syntax (GraalVM JS)
3. Access Java classes via `Java.type('org.gms.ClassName')`
4. Extend `AbstractPlayerInteraction` for player interaction APIs
5. Scripts are cached per-client, reload on client reconnect

### Working with Game Data (WZ)

- WZ data files should be in `wz/` or `wz-{language}/` directories
- Data loaded at server startup by factory classes
- Item data: `ItemInformationProvider`
- Monster data: `MonsterInformationProvider`
- Skill data: `SkillFactory`
- Map data: `MapFactory` (lazy-loaded)

## Common Configuration

### Database Connection

Edit `gms-server/src/main/resources/application.yml`:
```yaml
mybatis-flex:
  datasource:
    mysql:
      url: jdbc:mysql://localhost:3306/beidou?...
      username: root
      password: your_password
```

### Server Ports

```yaml
server:
  port: 8686  # REST API port

gms:
  service:
    login-port: 8484  # Game login port
    wan-host: 127.0.0.1  # External IP
    lan-host: 127.0.0.1  # Internal IP
```

### JWT Secret

**IMPORTANT:** Change JWT secret in production:
```yaml
jwt:
  secret: "your-uuid-here"  # Generate new UUID for production
  duration: 1800000  # 30 minutes in ms
```

### Swagger UI

Disable in production:
```yaml
springdoc:
  api-docs:
    enabled: false  # Disables OpenAPI
  swagger-ui:
    enabled: false  # Disables Swagger UI
```

## Testing

- Test files in `gms-server/src/test/java/`
- Includes utility classes: `CodeGen.java` (MyBatis-Flex code generation), `XmlDiff.java`, `XmlSort.java`
- Run tests with `mvn test`

## External Resources

- Client/server releases: https://github.com/BeiDouMS/BeiDou-Server/releases
- Wiki: https://github.com/BeiDouMS/BeiDou-Server/wiki
- Docker deployment: https://github.com/BeiDouMS/BeiDou-docker
- Asset API: https://maplestory.io (used by frontend for images)

## Important Notes

- **MySQL 8+ required** (MySQL 5.x not supported)
- **Java 21 required** (uses virtual threads and modern APIs)
- **Database auto-created** on first run if not exists
- **Multi-language support**: Scripts and WZ data use language-specific directories
- **Rate limiting**: Configurable per-IP rate limiting in `application.yml`
- **Main class**: `org.gms.ServerApplication` (Spring Boot entry point)
