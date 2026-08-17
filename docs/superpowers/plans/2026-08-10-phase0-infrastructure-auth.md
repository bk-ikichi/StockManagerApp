# フェーズ0：基盤構築（インフラ・認証・ログイン画面） 実装計画

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**目的：** 在庫管理アプリのデプロイ可能な骨組み（VPS＋Docker Compose＋Spring Boot＋PostgreSQL＋React）を立ち上げ、3種の共通アカウント（管理者/社員/ゲスト）でセッションベースログインができる状態（管理者の初回パスワード変更強制を含む）にする。以降の全フェーズはこの土台の上に機能を積み上げる。

**アーキテクチャ：** Spring Bootモノリスが、ビルド済みReact SPAを静的リソースとして同梱配信し、PostgreSQLをバックエンドに持つ。全体を1台のVPS上でDocker Composeにより稼働させる。認証はSpring Securityによるセッションベース（JWTは使わない）。詳細は`docs/superpowers/specs/2026-08-10-inventory-management-architecture-design.md`を参照。

**技術スタック：** Java 21、Spring Boot 3.3.4（Web／Security／MyBatis）、MyBatis（`mybatis-spring-boot-starter`）、PostgreSQL 16、Flyway、Maven、React 18＋TypeScript＋Vite、Vitest＋Testing Library、Docker Compose、Testcontainers（バックエンドの結合テスト）。

## 全体の制約

- バックエンド：Java/Spring Boot。フロントエンド：React（`在庫管理機能_設計ドキュメント_v8.md` 1章）
- 認証はセッションベース（Spring Security＋HttpSession）。JWTは使わない — 単一オリジン・単一インスタンス構成のためJWTの利点が活きず、共通アカウントのパスワード変更時にセッションを即座に無効化できる利点がある（アーキテクチャ設計書 §2.2）
- フロントエンドとバックエンドは同一オリジン：Reactの本番ビルドはSpring Bootの静的リソースとして配信し、別ホストには分離しない（アーキテクチャ設計書 §2.1）
- 主要テーブルには初回リリース時点から`store_id`を持たせる（アーキテクチャ設計書 §2.4）
- 店舗ごとの共通アカウントは3種：`ADMIN`（管理者/MG）、`STAFF`（社員）、`GUEST`（ゲスト）。個人ごとのアカウントではない（v8 10章）
- 初回ログイン時のパスワード変更が強制されるのは`ADMIN`のみ。`STAFF`/`GUEST`のパスワードは退職者が出るまで使い回す（v8 10章）
- ジョブスケジューラ（Quartz等）は導入しない。フェーズ0の処理はすべてリクエスト駆動（アーキテクチャ設計書 §2.3）
- **バックエンドのレイヤー構成は `mapper → repository → service → controller` の順。mapperはSQLを直接書くinterface（MyBatis）とし、Spring Data JPA／Hibernateは使わない。controllerの入出力はDTOを介する**（アーキテクチャ設計書 §2.5）
- **パッケージ構成はレイヤー別ディレクトリとする**：`mapper/`・`repository/`・`service/`・`controller/`・`dto/`はそれぞれの層のクラスのみを置く。認証・アカウント関連でどの層にも明確に属さないもの（Security設定、`UserDetailsService`実装、ドメインオブジェクト、例外クラスなど）は`auth/`にまとめる（2026-08-10、フェーズ0計画レビュー時に確定）
- フェーズ0のデプロイ先：開発中はConoHa VPS（アーキテクチャ設計書 §3.1）

---

## ファイル構成

```
backend/
  pom.xml
  src/main/java/com/bkikichi/stockmanager/
    StockManagerApplication.java
    mapper/
      StoreMapper.java                (mapper層：SQLを書くinterface)
      AccountMapper.java
    repository/
      StoreRepository.java            (repository層：mapperをラップ)
      AccountRepository.java
    service/
      AccountService.java             (service層：業務ロジック)
    controller/
      HealthController.java
      AuthController.java             (controller層。DTOのみを扱う)
    dto/
      LoginRequest.java
      AccountResponse.java
      ChangePasswordRequest.java
      MessageResponse.java
    auth/
      Store.java                      (ドメインオブジェクト。mapperが返す型)
      Account.java
      AccountRole.java
      SecurityConfig.java
      AccountUserDetailsService.java  (Spring Securityとの接続。service層に依存)
      InitialAccountSeeder.java       (起動時の初期アカウント投入。service層に依存)
      PasswordEncoderConfig.java
    exception/
      InvalidCurrentPasswordException.java
      WeakPasswordException.java
  src/main/resources/
    application.yml
    db/migration/
      V1__create_store_table.sql
      V2__create_account_table.sql
    static/            (Dockerビルド時にReactの本番ビルドが置かれる。ソース管理上は空)
  src/test/java/com/bkikichi/stockmanager/
    controller/
      HealthControllerTest.java
      AuthControllerTest.java
    migration/
      FlywayMigrationTest.java
    mapper/
      StoreMapperTest.java
      AccountMapperTest.java
    repository/
      StoreRepositoryTest.java
      AccountRepositoryTest.java
    service/
      AccountServiceTest.java

frontend/
  package.json
  vite.config.ts
  tsconfig.json
  index.html
  src/
    main.tsx
    App.tsx
    api/
      authApi.ts
    pages/
      LoginPage.tsx
      LoginPage.test.tsx
      ChangePasswordPage.tsx
      ChangePasswordPage.test.tsx
      HomePage.tsx
      HomePage.test.tsx
    test/
      setup.ts

Dockerfile
docker-compose.yml
.env.example
```

---

### タスク1：VPS準備とDocker動作確認

**ファイル：** なし（インフラ作業。リポジトリへの変更なし）

**インターフェース：**
- 成果物：`<VPS_IP>`でSSH接続可能、Docker/Docker Composeがインストール済みのVPS。タスク10のデプロイ手順で使用する。

このタスクは手動作業です。有料VPSへのサインアップや支払い情報の入力はコーディングエージェントには実行できません。手順1〜4は人間が行い、手順5はSSH接続後にエージェントが検証できます。

- [ ] **手順1（手動・人間が行う）：ConoHa VPSに申し込み、インスタンスを作成する**

2GBクラスのインスタンス（Ubuntu 22.04 LTS推奨）を作成し、パブリックIPアドレスを`<VPS_IP>`として控える。

- [ ] **手順2（手動・人間が行う）：SSH公開鍵をインスタンスに登録する**

ConoHaのコントロールパネルから`~/.ssh/id_ed25519.pub`（または専用のデプロイ鍵）を登録し、パスワードなしでSSHログインできるようにする。

- [ ] **手順3（手動・人間が行う）：VPS上にDocker／Docker Composeをインストールする**

SSHでログインし、以下を実行する。
```bash
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
```
グループ変更を反映させるため、一度ログアウト・再ログインする。最近のDockerには`docker compose`サブコマンドが標準で含まれるため、個別インストールは不要。

- [ ] **手順4（手動・人間が行う）：ConoHaのファイアウォール設定でポート8080を開放する**

ConoHaコントロールパネルのパケットフィルタ設定で、SSH用ポートに加えてTCP 8080番（アプリのポート、タスク10で使用）を全許可にする。

- [ ] **手順5：SSH接続とDockerの動作を確認する**

実行：`ssh <user>@<VPS_IP> "docker --version && docker compose version"`
期待結果：両コマンドがエラーなくバージョン番号を出力する。

---

### タスク2：バックエンド雛形とヘルスチェックAPI

**ファイル：**
- 作成：`backend/pom.xml`
- 作成：`backend/src/main/java/com/bkikichi/stockmanager/StockManagerApplication.java`
- 作成：`backend/src/main/java/com/bkikichi/stockmanager/controller/HealthController.java`
- 作成：`backend/src/main/resources/application.yml`
- テスト：`backend/src/test/java/com/bkikichi/stockmanager/controller/HealthControllerTest.java`

**インターフェース：**
- 成果物：`GET /api/health` → `200 {"status":"ok"}`。タスク10のデプロイ確認で使用する。

- [ ] **手順1：`backend/pom.xml`を作成する**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.3.4</version>
        <relativePath/>
    </parent>

    <groupId>com.bkikichi</groupId>
    <artifactId>stockmanager</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>stockmanager</name>
    <description>Stock management app backend</description>

    <properties>
        <java.version>21</java.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```

補足：この時点ではDB・MyBatis・Securityの依存関係はまだ追加しない。必要になったタスク（DB系はタスク3、MyBatisはタスク4、Securityはタスク6）でその都度追加する。

- [ ] **手順2：`backend/src/main/resources/application.yml`を作成する**

```yaml
spring:
  application:
    name: stockmanager

server:
  port: 8080
```

- [ ] **手順3：失敗するテストを書く**

`backend/src/test/java/com/bkikichi/stockmanager/controller/HealthControllerTest.java`：
```java
package com.bkikichi.stockmanager.controller;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.web.servlet.MockMvc;

import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.content;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;

@SpringBootTest
@AutoConfigureMockMvc
class HealthControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @Test
    void healthEndpointReturnsOkStatus() throws Exception {
        mockMvc.perform(get("/api/health"))
                .andExpect(status().isOk())
                .andExpect(content().json("{\"status\":\"ok\"}"));
    }
}
```

- [ ] **手順4：テストを実行し、失敗することを確認する**

実行：`cd backend && mvn -q -Dtest=HealthControllerTest test`
期待結果：FAIL（`HealthController`が存在せずコンパイルエラー）

- [ ] **手順5：最小限の実装を書く**

`backend/src/main/java/com/bkikichi/stockmanager/StockManagerApplication.java`：
```java
package com.bkikichi.stockmanager;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class StockManagerApplication {
    public static void main(String[] args) {
        SpringApplication.run(StockManagerApplication.class, args);
    }
}
```

`backend/src/main/java/com/bkikichi/stockmanager/controller/HealthController.java`：
```java
package com.bkikichi.stockmanager.controller;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

import java.util.Map;

@RestController
public class HealthController {

    @GetMapping("/api/health")
    public Map<String, String> health() {
        return Map.of("status", "ok");
    }
}
```

- [ ] **手順6：テストを実行し、成功することを確認する**

実行：`cd backend && mvn -q -Dtest=HealthControllerTest test`
期待結果：PASS

- [ ] **手順7：コミット**

```bash
git add backend/pom.xml backend/src/main/java/com/bkikichi/stockmanager/StockManagerApplication.java backend/src/main/java/com/bkikichi/stockmanager/controller/HealthController.java backend/src/main/resources/application.yml backend/src/test/java/com/bkikichi/stockmanager/controller/HealthControllerTest.java
git commit -m "feat(backend): Spring Bootアプリの雛形とヘルスチェックAPIを追加"
```

---

### タスク3：PostgreSQL + Flywayマイグレーション（`stores`／`accounts`テーブル）

**ファイル：**
- 変更：`backend/pom.xml`
- 変更：`backend/src/main/resources/application.yml`
- 作成：`backend/src/main/resources/db/migration/V1__create_store_table.sql`
- 作成：`backend/src/main/resources/db/migration/V2__create_account_table.sql`
- テスト：`backend/src/test/java/com/bkikichi/stockmanager/migration/FlywayMigrationTest.java`

**インターフェース：**
- 成果物：`stores`テーブル（`id`、`name`、`created_at`）と`accounts`テーブル（`id`、`store_id`（外部キー→stores）、`username`（一意）、`password_hash`、`role`（CHECK制約 'ADMIN'/'STAFF'/'GUEST'）、`must_change_password`、`created_at`、`updated_at`）。タスク4のmapper層が使用する。

前提：Testcontainersがテストごとに実際のPostgreSQLコンテナを起動するため、ローカルでDockerが起動している必要がある。

- [ ] **手順1：`backend/pom.xml`を変更し、DB・マイグレーション・テスト用の依存関係を追加する**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.3.4</version>
        <relativePath/>
    </parent>

    <groupId>com.bkikichi</groupId>
    <artifactId>stockmanager</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>stockmanager</name>
    <description>Stock management app backend</description>

    <properties>
        <java.version>21</java.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-jdbc</artifactId>
        </dependency>
        <dependency>
            <groupId>org.postgresql</groupId>
            <artifactId>postgresql</artifactId>
            <scope>runtime</scope>
        </dependency>
        <dependency>
            <groupId>org.flywaydb</groupId>
            <artifactId>flyway-core</artifactId>
        </dependency>
        <dependency>
            <groupId>org.flywaydb</groupId>
            <artifactId>flyway-database-postgresql</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.testcontainers</groupId>
            <artifactId>junit-jupiter</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.testcontainers</groupId>
            <artifactId>postgresql</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.testcontainers</groupId>
                <artifactId>testcontainers-bom</artifactId>
                <version>1.20.1</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```

- [ ] **手順2：`backend/src/main/resources/application.yml`を変更し、DB接続とFlyway設定を追加する**

```yaml
spring:
  application:
    name: stockmanager
  datasource:
    url: jdbc:postgresql://${DB_HOST:localhost}:${DB_PORT:5432}/${DB_NAME:stockmanager}
    username: ${DB_USER:stockmanager}
    password: ${DB_PASSWORD:stockmanager}
  flyway:
    enabled: true
    locations: classpath:db/migration

server:
  port: 8080
```

- [ ] **手順3：失敗するテストを書く**

`backend/src/test/java/com/bkikichi/stockmanager/migration/FlywayMigrationTest.java`：
```java
package com.bkikichi.stockmanager.migration;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;
import org.testcontainers.containers.PostgreSQLContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;

import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

@Testcontainers
@SpringBootTest
class FlywayMigrationTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine");

    @DynamicPropertySource
    static void configureDatasource(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired
    private JdbcTemplate jdbcTemplate;

    @Test
    void storesTableExistsWithExpectedColumns() {
        List<String> columns = jdbcTemplate.queryForList(
                "SELECT column_name FROM information_schema.columns WHERE table_name = 'stores'",
                String.class);

        assertThat(columns).containsExactlyInAnyOrder("id", "name", "created_at");
    }

    @Test
    void accountsTableExistsWithExpectedColumnsAndForeignKey() {
        List<String> columns = jdbcTemplate.queryForList(
                "SELECT column_name FROM information_schema.columns WHERE table_name = 'accounts'",
                String.class);

        assertThat(columns).containsExactlyInAnyOrder(
                "id", "store_id", "username", "password_hash", "role",
                "must_change_password", "created_at", "updated_at");

        Long fkCount = jdbcTemplate.queryForObject(
                "SELECT count(*) FROM information_schema.table_constraints " +
                        "WHERE table_name = 'accounts' AND constraint_type = 'FOREIGN KEY'",
                Long.class);
        assertThat(fkCount).isEqualTo(1L);
    }
}
```

- [ ] **手順4：テストを実行し、失敗することを確認する**

実行：`cd backend && mvn -q -Dtest=FlywayMigrationTest test`
期待結果：FAIL（`relation "stores" does not exist`。マイグレーションがまだない）

- [ ] **手順5：マイグレーションを書く**

`backend/src/main/resources/db/migration/V1__create_store_table.sql`：
```sql
CREATE TABLE stores (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT now()
);
```

`backend/src/main/resources/db/migration/V2__create_account_table.sql`：
```sql
CREATE TABLE accounts (
    id BIGSERIAL PRIMARY KEY,
    store_id BIGINT NOT NULL REFERENCES stores(id),
    username VARCHAR(50) NOT NULL UNIQUE,
    password_hash VARCHAR(100) NOT NULL,
    role VARCHAR(20) NOT NULL CHECK (role IN ('ADMIN', 'STAFF', 'GUEST')),
    must_change_password BOOLEAN NOT NULL DEFAULT false,
    created_at TIMESTAMP NOT NULL DEFAULT now(),
    updated_at TIMESTAMP NOT NULL DEFAULT now()
);
```

- [ ] **手順6：テストを実行し、成功することを確認する**

実行：`cd backend && mvn -q -Dtest=FlywayMigrationTest test`
期待結果：PASS

- [ ] **手順7：コミット**

```bash
git add backend/pom.xml backend/src/main/resources/application.yml backend/src/main/resources/db/migration backend/src/test/java/com/bkikichi/stockmanager/migration
git commit -m "feat(backend): stores/accountsテーブルのFlywayマイグレーションを追加"
```

---

### タスク4：Mapper層（`StoreMapper`／`AccountMapper`）

**ファイル：**
- 変更：`backend/pom.xml`
- 変更：`backend/src/main/resources/application.yml`
- 作成：`backend/src/main/java/com/bkikichi/stockmanager/auth/Store.java`
- 作成：`backend/src/main/java/com/bkikichi/stockmanager/mapper/StoreMapper.java`
- 作成：`backend/src/main/java/com/bkikichi/stockmanager/auth/Account.java`
- 作成：`backend/src/main/java/com/bkikichi/stockmanager/auth/AccountRole.java`
- 作成：`backend/src/main/java/com/bkikichi/stockmanager/mapper/AccountMapper.java`
- テスト：`backend/src/test/java/com/bkikichi/stockmanager/mapper/StoreMapperTest.java`
- テスト：`backend/src/test/java/com/bkikichi/stockmanager/mapper/AccountMapperTest.java`

**インターフェース：**
- 消費：タスク3の`stores`／`accounts`テーブル。
- 成果物：`Store`／`Account`ドメインオブジェクト（`auth`パッケージ。getter/setter付きの素のPOJO、ORMアノテーションは持たない）。`StoreMapper.insert(Store)`／`findById(Long): Store`／`count(): long`。`AccountMapper.insert(Account)`／`findByUsername(String): Account`（見つからない場合は`null`）／`updatePassword(Account)`／`count(): long`。タスク5のrepository層が使用する。

- [ ] **手順1：`backend/pom.xml`を変更し、MyBatisを追加する**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.3.4</version>
        <relativePath/>
    </parent>

    <groupId>com.bkikichi</groupId>
    <artifactId>stockmanager</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>stockmanager</name>
    <description>Stock management app backend</description>

    <properties>
        <java.version>21</java.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-jdbc</artifactId>
        </dependency>
        <dependency>
            <groupId>org.mybatis.spring.boot</groupId>
            <artifactId>mybatis-spring-boot-starter</artifactId>
            <version>3.0.3</version>
        </dependency>
        <dependency>
            <groupId>org.postgresql</groupId>
            <artifactId>postgresql</artifactId>
            <scope>runtime</scope>
        </dependency>
        <dependency>
            <groupId>org.flywaydb</groupId>
            <artifactId>flyway-core</artifactId>
        </dependency>
        <dependency>
            <groupId>org.flywaydb</groupId>
            <artifactId>flyway-database-postgresql</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.testcontainers</groupId>
            <artifactId>junit-jupiter</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.testcontainers</groupId>
            <artifactId>postgresql</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.testcontainers</groupId>
                <artifactId>testcontainers-bom</artifactId>
                <version>1.20.1</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```

- [ ] **手順2：`backend/src/main/resources/application.yml`を変更し、MyBatisのキャメルケース変換を有効にする**

```yaml
spring:
  application:
    name: stockmanager
  datasource:
    url: jdbc:postgresql://${DB_HOST:localhost}:${DB_PORT:5432}/${DB_NAME:stockmanager}
    username: ${DB_USER:stockmanager}
    password: ${DB_PASSWORD:stockmanager}
  flyway:
    enabled: true
    locations: classpath:db/migration

mybatis:
  configuration:
    map-underscore-to-camel-case: true

server:
  port: 8080
```

- [ ] **手順3：失敗するテストを書く**

`backend/src/test/java/com/bkikichi/stockmanager/mapper/StoreMapperTest.java`：
```java
package com.bkikichi.stockmanager.mapper;

import com.bkikichi.stockmanager.auth.Store;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;
import org.testcontainers.containers.PostgreSQLContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;

import static org.assertj.core.api.Assertions.assertThat;

@Testcontainers
@SpringBootTest
class StoreMapperTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine");

    @DynamicPropertySource
    static void configureDatasource(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired
    private StoreMapper storeMapper;

    @Test
    void insertAssignsGeneratedIdAndFindByIdReturnsIt() {
        Store store = new Store("テスト店舗");
        storeMapper.insert(store);

        assertThat(store.getId()).isNotNull();

        Store found = storeMapper.findById(store.getId());
        assertThat(found.getName()).isEqualTo("テスト店舗");
    }

    @Test
    void countReflectsInsertedRows() {
        long before = storeMapper.count();
        storeMapper.insert(new Store("カウント用店舗"));

        assertThat(storeMapper.count()).isEqualTo(before + 1);
    }
}
```

`backend/src/test/java/com/bkikichi/stockmanager/mapper/AccountMapperTest.java`：
```java
package com.bkikichi.stockmanager.mapper;

import com.bkikichi.stockmanager.auth.Account;
import com.bkikichi.stockmanager.auth.AccountRole;
import com.bkikichi.stockmanager.auth.Store;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;
import org.testcontainers.containers.PostgreSQLContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;

import static org.assertj.core.api.Assertions.assertThat;

@Testcontainers
@SpringBootTest
class AccountMapperTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine");

    @DynamicPropertySource
    static void configureDatasource(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired
    private StoreMapper storeMapper;

    @Autowired
    private AccountMapper accountMapper;

    @Test
    void insertAndFindByUsernameRoundTripsAllFields() {
        Store store = new Store("テスト店舗");
        storeMapper.insert(store);

        Account account = new Account(store.getId(), "mapper-test-mg", "dummy-hash", AccountRole.ADMIN, true);
        accountMapper.insert(account);

        assertThat(account.getId()).isNotNull();

        Account found = accountMapper.findByUsername("mapper-test-mg");
        assertThat(found.getStoreId()).isEqualTo(store.getId());
        assertThat(found.getPasswordHash()).isEqualTo("dummy-hash");
        assertThat(found.getRole()).isEqualTo(AccountRole.ADMIN);
        assertThat(found.isMustChangePassword()).isTrue();
    }

    @Test
    void findByUsernameReturnsNullWhenNotFound() {
        assertThat(accountMapper.findByUsername("no-such-user")).isNull();
    }

    @Test
    void updatePasswordChangesHashAndFlag() {
        Store store = new Store("テスト店舗2");
        storeMapper.insert(store);
        Account account = new Account(store.getId(), "mapper-test-mg-2", "old-hash", AccountRole.ADMIN, true);
        accountMapper.insert(account);

        account.setPasswordHash("new-hash");
        account.setMustChangePassword(false);
        accountMapper.updatePassword(account);

        Account reloaded = accountMapper.findByUsername("mapper-test-mg-2");
        assertThat(reloaded.getPasswordHash()).isEqualTo("new-hash");
        assertThat(reloaded.isMustChangePassword()).isFalse();
    }
}
```

- [ ] **手順4：テストを実行し、失敗することを確認する**

実行：`cd backend && mvn -q -Dtest=StoreMapperTest,AccountMapperTest test`
期待結果：FAIL（`Store`、`StoreMapper`、`Account`、`AccountRole`、`AccountMapper`が存在せずコンパイルエラー）

- [ ] **手順5：最小限の実装を書く**

`backend/src/main/java/com/bkikichi/stockmanager/auth/Store.java`：
```java
package com.bkikichi.stockmanager.auth;

import java.time.Instant;

public class Store {

    private Long id;
    private String name;
    private Instant createdAt;

    public Store() {
    }

    public Store(String name) {
        this.name = name;
    }

    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public Instant getCreatedAt() {
        return createdAt;
    }

    public void setCreatedAt(Instant createdAt) {
        this.createdAt = createdAt;
    }
}
```

`backend/src/main/java/com/bkikichi/stockmanager/mapper/StoreMapper.java`：
```java
package com.bkikichi.stockmanager.mapper;

import com.bkikichi.stockmanager.auth.Store;
import org.apache.ibatis.annotations.Insert;
import org.apache.ibatis.annotations.Mapper;
import org.apache.ibatis.annotations.Options;
import org.apache.ibatis.annotations.Param;
import org.apache.ibatis.annotations.Select;

@Mapper
public interface StoreMapper {

    @Insert("INSERT INTO stores (name) VALUES (#{name})")
    @Options(useGeneratedKeys = true, keyProperty = "id")
    void insert(Store store);

    @Select("SELECT id, name, created_at FROM stores WHERE id = #{id}")
    Store findById(@Param("id") Long id);

    @Select("SELECT count(*) FROM stores")
    long count();
}
```

`backend/src/main/java/com/bkikichi/stockmanager/auth/AccountRole.java`：
```java
package com.bkikichi.stockmanager.auth;

public enum AccountRole {
    ADMIN,
    STAFF,
    GUEST
}
```

`backend/src/main/java/com/bkikichi/stockmanager/auth/Account.java`：
```java
package com.bkikichi.stockmanager.auth;

import java.time.Instant;

public class Account {

    private Long id;
    private Long storeId;
    private String username;
    private String passwordHash;
    private AccountRole role;
    private boolean mustChangePassword;
    private Instant createdAt;
    private Instant updatedAt;

    public Account() {
    }

    public Account(Long storeId, String username, String passwordHash, AccountRole role, boolean mustChangePassword) {
        this.storeId = storeId;
        this.username = username;
        this.passwordHash = passwordHash;
        this.role = role;
        this.mustChangePassword = mustChangePassword;
    }

    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }

    public Long getStoreId() {
        return storeId;
    }

    public void setStoreId(Long storeId) {
        this.storeId = storeId;
    }

    public String getUsername() {
        return username;
    }

    public void setUsername(String username) {
        this.username = username;
    }

    public String getPasswordHash() {
        return passwordHash;
    }

    public void setPasswordHash(String passwordHash) {
        this.passwordHash = passwordHash;
    }

    public AccountRole getRole() {
        return role;
    }

    public void setRole(AccountRole role) {
        this.role = role;
    }

    public boolean isMustChangePassword() {
        return mustChangePassword;
    }

    public void setMustChangePassword(boolean mustChangePassword) {
        this.mustChangePassword = mustChangePassword;
    }

    public Instant getCreatedAt() {
        return createdAt;
    }

    public void setCreatedAt(Instant createdAt) {
        this.createdAt = createdAt;
    }

    public Instant getUpdatedAt() {
        return updatedAt;
    }

    public void setUpdatedAt(Instant updatedAt) {
        this.updatedAt = updatedAt;
    }
}
```

`backend/src/main/java/com/bkikichi/stockmanager/mapper/AccountMapper.java`：
```java
package com.bkikichi.stockmanager.mapper;

import com.bkikichi.stockmanager.auth.Account;
import org.apache.ibatis.annotations.Insert;
import org.apache.ibatis.annotations.Mapper;
import org.apache.ibatis.annotations.Options;
import org.apache.ibatis.annotations.Param;
import org.apache.ibatis.annotations.Select;
import org.apache.ibatis.annotations.Update;

@Mapper
public interface AccountMapper {

    @Insert("INSERT INTO accounts (store_id, username, password_hash, role, must_change_password) " +
            "VALUES (#{storeId}, #{username}, #{passwordHash}, #{role}, #{mustChangePassword})")
    @Options(useGeneratedKeys = true, keyProperty = "id")
    void insert(Account account);

    @Select("SELECT id, store_id, username, password_hash, role, must_change_password, created_at, updated_at " +
            "FROM accounts WHERE username = #{username}")
    Account findByUsername(@Param("username") String username);

    @Update("UPDATE accounts SET password_hash = #{passwordHash}, must_change_password = #{mustChangePassword}, " +
            "updated_at = now() WHERE id = #{id}")
    void updatePassword(Account account);

    @Select("SELECT count(*) FROM accounts")
    long count();
}
```

- [ ] **手順6：テストを実行し、成功することを確認する**

実行：`cd backend && mvn -q -Dtest=StoreMapperTest,AccountMapperTest test`
期待結果：PASS

- [ ] **手順7：コミット**

```bash
git add backend/pom.xml backend/src/main/resources/application.yml backend/src/main/java/com/bkikichi/stockmanager/auth/Store.java backend/src/main/java/com/bkikichi/stockmanager/auth/Account.java backend/src/main/java/com/bkikichi/stockmanager/auth/AccountRole.java backend/src/main/java/com/bkikichi/stockmanager/mapper backend/src/test/java/com/bkikichi/stockmanager/mapper
git commit -m "feat(backend): StoreMapper/AccountMapperをMyBatisで追加"
```

---

### タスク5：Repository層（`StoreRepository`／`AccountRepository`）

**ファイル：**
- 作成：`backend/src/main/java/com/bkikichi/stockmanager/repository/StoreRepository.java`
- 作成：`backend/src/main/java/com/bkikichi/stockmanager/repository/AccountRepository.java`
- テスト：`backend/src/test/java/com/bkikichi/stockmanager/repository/StoreRepositoryTest.java`
- テスト：`backend/src/test/java/com/bkikichi/stockmanager/repository/AccountRepositoryTest.java`

**インターフェース：**
- 消費：タスク4の`StoreMapper`／`AccountMapper`。
- 成果物：`StoreRepository.save(Store): Store`／`findById(Long): Store`。`AccountRepository.save(Account): Account`／`findByUsername(String): Optional<Account>`／`updatePassword(Account)`／`count(): long`。タスク6のservice層が使用する。

- [ ] **手順1：失敗するテストを書く**

`backend/src/test/java/com/bkikichi/stockmanager/repository/StoreRepositoryTest.java`：
```java
package com.bkikichi.stockmanager.repository;

import com.bkikichi.stockmanager.auth.Store;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;
import org.testcontainers.containers.PostgreSQLContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;

import static org.assertj.core.api.Assertions.assertThat;

@Testcontainers
@SpringBootTest
class StoreRepositoryTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine");

    @DynamicPropertySource
    static void configureDatasource(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired
    private StoreRepository storeRepository;

    @Test
    void saveReturnsStoreWithGeneratedId() {
        Store saved = storeRepository.save(new Store("リポジトリテスト店舗"));

        assertThat(saved.getId()).isNotNull();
        assertThat(storeRepository.findById(saved.getId()).getName()).isEqualTo("リポジトリテスト店舗");
    }
}
```

`backend/src/test/java/com/bkikichi/stockmanager/repository/AccountRepositoryTest.java`：
```java
package com.bkikichi.stockmanager.repository;

import com.bkikichi.stockmanager.auth.Account;
import com.bkikichi.stockmanager.auth.AccountRole;
import com.bkikichi.stockmanager.auth.Store;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;
import org.testcontainers.containers.PostgreSQLContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;

import static org.assertj.core.api.Assertions.assertThat;

@Testcontainers
@SpringBootTest
class AccountRepositoryTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine");

    @DynamicPropertySource
    static void configureDatasource(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired
    private StoreRepository storeRepository;

    @Autowired
    private AccountRepository accountRepository;

    @Test
    void savesAndFindsAccountByUsername() {
        Store store = storeRepository.save(new Store("テスト店舗"));
        accountRepository.save(new Account(store.getId(), "repo-test-mg",
                "dummy-hash", AccountRole.ADMIN, true));

        Account found = accountRepository.findByUsername("repo-test-mg").orElseThrow();

        assertThat(found.getStoreId()).isEqualTo(store.getId());
        assertThat(found.getRole()).isEqualTo(AccountRole.ADMIN);
    }

    @Test
    void returnsEmptyOptionalWhenUsernameNotFound() {
        assertThat(accountRepository.findByUsername("nonexistent")).isEmpty();
    }

    @Test
    void updatePasswordPersistsNewHash() {
        Store store = storeRepository.save(new Store("テスト店舗2"));
        Account account = accountRepository.save(new Account(store.getId(), "repo-test-mg-2",
                "old-hash", AccountRole.ADMIN, true));

        account.setPasswordHash("new-hash");
        account.setMustChangePassword(false);
        accountRepository.updatePassword(account);

        Account reloaded = accountRepository.findByUsername("repo-test-mg-2").orElseThrow();
        assertThat(reloaded.getPasswordHash()).isEqualTo("new-hash");
        assertThat(reloaded.isMustChangePassword()).isFalse();
    }
}
```

- [ ] **手順2：テストを実行し、失敗することを確認する**

実行：`cd backend && mvn -q -Dtest=StoreRepositoryTest,AccountRepositoryTest test`
期待結果：FAIL（`StoreRepository`、`AccountRepository`が存在せずコンパイルエラー）

- [ ] **手順3：最小限の実装を書く**

`backend/src/main/java/com/bkikichi/stockmanager/repository/StoreRepository.java`：
```java
package com.bkikichi.stockmanager.repository;

import com.bkikichi.stockmanager.auth.Store;
import com.bkikichi.stockmanager.mapper.StoreMapper;
import org.springframework.stereotype.Repository;

@Repository
public class StoreRepository {

    private final StoreMapper storeMapper;

    public StoreRepository(StoreMapper storeMapper) {
        this.storeMapper = storeMapper;
    }

    public Store save(Store store) {
        storeMapper.insert(store);
        return store;
    }

    public Store findById(Long id) {
        return storeMapper.findById(id);
    }

    public long count() {
        return storeMapper.count();
    }
}
```

`backend/src/main/java/com/bkikichi/stockmanager/repository/AccountRepository.java`：
```java
package com.bkikichi.stockmanager.repository;

import com.bkikichi.stockmanager.auth.Account;
import com.bkikichi.stockmanager.mapper.AccountMapper;
import org.springframework.stereotype.Repository;

import java.util.Optional;

@Repository
public class AccountRepository {

    private final AccountMapper accountMapper;

    public AccountRepository(AccountMapper accountMapper) {
        this.accountMapper = accountMapper;
    }

    public Account save(Account account) {
        accountMapper.insert(account);
        return account;
    }

    public Optional<Account> findByUsername(String username) {
        return Optional.ofNullable(accountMapper.findByUsername(username));
    }

    public void updatePassword(Account account) {
        accountMapper.updatePassword(account);
    }

    public long count() {
        return accountMapper.count();
    }
}
```

- [ ] **手順4：テストを実行し、成功することを確認する**

実行：`cd backend && mvn -q -Dtest=StoreRepositoryTest,AccountRepositoryTest test`
期待結果：PASS

- [ ] **手順5：コミット**

```bash
git add backend/src/main/java/com/bkikichi/stockmanager/repository backend/src/test/java/com/bkikichi/stockmanager/repository
git commit -m "feat(backend): StoreRepository/AccountRepositoryを追加"
```

---

### タスク6：Service層（`AccountService`）とパスワード処理・初期アカウント投入

**ファイル：**
- 変更：`backend/pom.xml`
- 作成：`backend/src/main/java/com/bkikichi/stockmanager/auth/PasswordEncoderConfig.java`
- 作成：`backend/src/main/java/com/bkikichi/stockmanager/exception/InvalidCurrentPasswordException.java`
- 作成：`backend/src/main/java/com/bkikichi/stockmanager/exception/WeakPasswordException.java`
- 作成：`backend/src/main/java/com/bkikichi/stockmanager/service/AccountService.java`
- 作成：`backend/src/main/java/com/bkikichi/stockmanager/auth/AccountUserDetailsService.java`
- 作成：`backend/src/main/java/com/bkikichi/stockmanager/auth/InitialAccountSeeder.java`
- テスト：`backend/src/test/java/com/bkikichi/stockmanager/service/AccountServiceTest.java`

**インターフェース：**
- 消費：タスク5の`StoreRepository`／`AccountRepository`。
- 成果物：`AccountService.findByUsername(String): Optional<Account>`／`seedInitialAccountsIfNeeded()`／`changePassword(String username, String currentPassword, String newPassword)`（失敗時は`InvalidCurrentPasswordException`または`WeakPasswordException`をスロー）。タスク7のcontroller層が使用する。アプリ起動時に`mg`（ADMIN、要パスワード変更）、`staff`（STAFF）、`guest`（GUEST）の3アカウントが自動投入される。

- [ ] **手順1：`backend/pom.xml`を変更し、Spring Security（パスワードハッシュ化用）を追加する**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.3.4</version>
        <relativePath/>
    </parent>

    <groupId>com.bkikichi</groupId>
    <artifactId>stockmanager</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>stockmanager</name>
    <description>Stock management app backend</description>

    <properties>
        <java.version>21</java.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-security</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-jdbc</artifactId>
        </dependency>
        <dependency>
            <groupId>org.mybatis.spring.boot</groupId>
            <artifactId>mybatis-spring-boot-starter</artifactId>
            <version>3.0.3</version>
        </dependency>
        <dependency>
            <groupId>org.postgresql</groupId>
            <artifactId>postgresql</artifactId>
            <scope>runtime</scope>
        </dependency>
        <dependency>
            <groupId>org.flywaydb</groupId>
            <artifactId>flyway-core</artifactId>
        </dependency>
        <dependency>
            <groupId>org.flywaydb</groupId>
            <artifactId>flyway-database-postgresql</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.springframework.security</groupId>
            <artifactId>spring-security-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.testcontainers</groupId>
            <artifactId>junit-jupiter</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.testcontainers</groupId>
            <artifactId>postgresql</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.testcontainers</groupId>
                <artifactId>testcontainers-bom</artifactId>
                <version>1.20.1</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```

補足：`spring-boot-starter-security`を追加した時点でSpring Bootのデフォルトのセキュリティ自動設定が有効になり、`/api/health`を含む全エンドポイントが一時的に認証必須になる（デフォルトパスワードが起動ログに出力される）。この状態はタスク7で`SecurityConfig`を書くまでの一時的なものなので、タスク6の完了時点では`HealthControllerTest`（タスク2）が失敗するようになる。これは想定内であり、タスク7で解消される。

- [ ] **手順2：失敗するテストを書く**

`backend/src/test/java/com/bkikichi/stockmanager/service/AccountServiceTest.java`：
```java
package com.bkikichi.stockmanager.service;

import com.bkikichi.stockmanager.auth.Account;
import com.bkikichi.stockmanager.auth.AccountRole;
import com.bkikichi.stockmanager.auth.Store;
import com.bkikichi.stockmanager.exception.InvalidCurrentPasswordException;
import com.bkikichi.stockmanager.exception.WeakPasswordException;
import com.bkikichi.stockmanager.repository.AccountRepository;
import com.bkikichi.stockmanager.repository.StoreRepository;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;
import org.springframework.security.crypto.password.PasswordEncoder;

import java.util.Optional;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.ArgumentMatchers.anyString;
import static org.mockito.Mockito.never;
import static org.mockito.Mockito.times;
import static org.mockito.Mockito.verify;
import static org.mockito.Mockito.when;

@ExtendWith(MockitoExtension.class)
class AccountServiceTest {

    @Mock
    private StoreRepository storeRepository;

    @Mock
    private AccountRepository accountRepository;

    @Mock
    private PasswordEncoder passwordEncoder;

    @Test
    void seedInitialAccountsIfNeededDoesNothingWhenAccountsAlreadyExist() {
        when(accountRepository.count()).thenReturn(3L);
        AccountService service = new AccountService(storeRepository, accountRepository, passwordEncoder);

        service.seedInitialAccountsIfNeeded();

        verify(storeRepository, never()).save(any());
    }

    @Test
    void seedInitialAccountsIfNeededCreatesStoreAndThreeAccountsWhenEmpty() {
        when(accountRepository.count()).thenReturn(0L);
        Store store = new Store("1号店");
        store.setId(1L);
        when(storeRepository.save(any(Store.class))).thenReturn(store);
        when(passwordEncoder.encode(anyString())).thenReturn("encoded");
        AccountService service = new AccountService(storeRepository, accountRepository, passwordEncoder);

        service.seedInitialAccountsIfNeeded();

        verify(accountRepository, times(3)).save(any(Account.class));
    }

    @Test
    void changePasswordThrowsWhenCurrentPasswordIsWrong() {
        Account account = new Account(1L, "mg", "old-hash", AccountRole.ADMIN, true);
        when(accountRepository.findByUsername("mg")).thenReturn(Optional.of(account));
        when(passwordEncoder.matches("wrong", "old-hash")).thenReturn(false);
        AccountService service = new AccountService(storeRepository, accountRepository, passwordEncoder);

        assertThatThrownBy(() -> service.changePassword("mg", "wrong", "NewPass123!"))
                .isInstanceOf(InvalidCurrentPasswordException.class);
    }

    @Test
    void changePasswordThrowsWhenNewPasswordIsTooShort() {
        Account account = new Account(1L, "mg", "old-hash", AccountRole.ADMIN, true);
        when(accountRepository.findByUsername("mg")).thenReturn(Optional.of(account));
        when(passwordEncoder.matches("old-plain", "old-hash")).thenReturn(true);
        AccountService service = new AccountService(storeRepository, accountRepository, passwordEncoder);

        assertThatThrownBy(() -> service.changePassword("mg", "old-plain", "short"))
                .isInstanceOf(WeakPasswordException.class);
    }

    @Test
    void changePasswordUpdatesHashAndClearsFlagOnSuccess() {
        Account account = new Account(1L, "mg", "old-hash", AccountRole.ADMIN, true);
        when(accountRepository.findByUsername("mg")).thenReturn(Optional.of(account));
        when(passwordEncoder.matches("old-plain", "old-hash")).thenReturn(true);
        when(passwordEncoder.encode("NewPass123!")).thenReturn("new-hash");
        AccountService service = new AccountService(storeRepository, accountRepository, passwordEncoder);

        service.changePassword("mg", "old-plain", "NewPass123!");

        assertThat(account.getPasswordHash()).isEqualTo("new-hash");
        assertThat(account.isMustChangePassword()).isFalse();
        verify(accountRepository).updatePassword(account);
    }
}
```

- [ ] **手順3：テストを実行し、失敗することを確認する**

実行：`cd backend && mvn -q -Dtest=AccountServiceTest test`
期待結果：FAIL（`AccountService`、`InvalidCurrentPasswordException`、`WeakPasswordException`が存在せずコンパイルエラー）

- [ ] **手順4：最小限の実装を書く**

`backend/src/main/java/com/bkikichi/stockmanager/auth/PasswordEncoderConfig.java`：
```java
package com.bkikichi.stockmanager.auth;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;

@Configuration
public class PasswordEncoderConfig {

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

`backend/src/main/java/com/bkikichi/stockmanager/exception/InvalidCurrentPasswordException.java`：
```java
package com.bkikichi.stockmanager.exception;

public class InvalidCurrentPasswordException extends RuntimeException {
    public InvalidCurrentPasswordException() {
        super("現在のパスワードが正しくありません");
    }
}
```

`backend/src/main/java/com/bkikichi/stockmanager/exception/WeakPasswordException.java`：
```java
package com.bkikichi.stockmanager.exception;

public class WeakPasswordException extends RuntimeException {
    public WeakPasswordException() {
        super("新しいパスワードは8文字以上で入力してください");
    }
}
```

`backend/src/main/java/com/bkikichi/stockmanager/service/AccountService.java`：
```java
package com.bkikichi.stockmanager.service;

import com.bkikichi.stockmanager.auth.Account;
import com.bkikichi.stockmanager.auth.AccountRole;
import com.bkikichi.stockmanager.auth.Store;
import com.bkikichi.stockmanager.exception.InvalidCurrentPasswordException;
import com.bkikichi.stockmanager.exception.WeakPasswordException;
import com.bkikichi.stockmanager.repository.AccountRepository;
import com.bkikichi.stockmanager.repository.StoreRepository;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.stereotype.Service;

import java.util.Optional;

@Service
public class AccountService {

    private final StoreRepository storeRepository;
    private final AccountRepository accountRepository;
    private final PasswordEncoder passwordEncoder;

    public AccountService(StoreRepository storeRepository,
                           AccountRepository accountRepository,
                           PasswordEncoder passwordEncoder) {
        this.storeRepository = storeRepository;
        this.accountRepository = accountRepository;
        this.passwordEncoder = passwordEncoder;
    }

    public Optional<Account> findByUsername(String username) {
        return accountRepository.findByUsername(username);
    }

    public void seedInitialAccountsIfNeeded() {
        if (accountRepository.count() > 0) {
            return;
        }

        Store store = storeRepository.save(new Store("1号店"));

        accountRepository.save(new Account(store.getId(), "mg",
                passwordEncoder.encode("ChangeMe123!"), AccountRole.ADMIN, true));
        accountRepository.save(new Account(store.getId(), "staff",
                passwordEncoder.encode("staff0000"), AccountRole.STAFF, false));
        accountRepository.save(new Account(store.getId(), "guest",
                passwordEncoder.encode("guest0000"), AccountRole.GUEST, false));
    }

    public void changePassword(String username, String currentPassword, String newPassword) {
        Account account = accountRepository.findByUsername(username)
                .orElseThrow(() -> new IllegalStateException("Unknown account: " + username));

        if (!passwordEncoder.matches(currentPassword, account.getPasswordHash())) {
            throw new InvalidCurrentPasswordException();
        }
        if (newPassword == null || newPassword.length() < 8) {
            throw new WeakPasswordException();
        }

        account.setPasswordHash(passwordEncoder.encode(newPassword));
        account.setMustChangePassword(false);
        accountRepository.updatePassword(account);
    }
}
```

`backend/src/main/java/com/bkikichi/stockmanager/auth/AccountUserDetailsService.java`：
```java
package com.bkikichi.stockmanager.auth;

import com.bkikichi.stockmanager.service.AccountService;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.core.userdetails.User;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.core.userdetails.UsernameNotFoundException;
import org.springframework.stereotype.Service;

@Service
public class AccountUserDetailsService implements UserDetailsService {

    private final AccountService accountService;

    public AccountUserDetailsService(AccountService accountService) {
        this.accountService = accountService;
    }

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        Account account = accountService.findByUsername(username)
                .orElseThrow(() -> new UsernameNotFoundException("Unknown account: " + username));

        return User.withUsername(account.getUsername())
                .password(account.getPasswordHash())
                .authorities(new SimpleGrantedAuthority("ROLE_" + account.getRole().name()))
                .build();
    }
}
```

`backend/src/main/java/com/bkikichi/stockmanager/auth/InitialAccountSeeder.java`：
```java
package com.bkikichi.stockmanager.auth;

import com.bkikichi.stockmanager.service.AccountService;
import org.springframework.boot.ApplicationArguments;
import org.springframework.boot.ApplicationRunner;
import org.springframework.stereotype.Component;

@Component
public class InitialAccountSeeder implements ApplicationRunner {

    private final AccountService accountService;

    public InitialAccountSeeder(AccountService accountService) {
        this.accountService = accountService;
    }

    @Override
    public void run(ApplicationArguments args) {
        accountService.seedInitialAccountsIfNeeded();
    }
}
```

- [ ] **手順5：テストを実行し、成功することを確認する**

実行：`cd backend && mvn -q -Dtest=AccountServiceTest test`
期待結果：PASS

- [ ] **手順6：コミット**

```bash
git add backend/pom.xml backend/src/main/java/com/bkikichi/stockmanager/auth/PasswordEncoderConfig.java backend/src/main/java/com/bkikichi/stockmanager/auth/AccountUserDetailsService.java backend/src/main/java/com/bkikichi/stockmanager/auth/InitialAccountSeeder.java backend/src/main/java/com/bkikichi/stockmanager/exception backend/src/main/java/com/bkikichi/stockmanager/service backend/src/test/java/com/bkikichi/stockmanager/service
git commit -m "feat(backend): AccountServiceと初期アカウント自動投入を追加"
```

---

### タスク7：Controller層（`AuthController`）とSpring Security設定

**ファイル：**
- 作成：`backend/src/main/java/com/bkikichi/stockmanager/auth/SecurityConfig.java`
- 作成：`backend/src/main/java/com/bkikichi/stockmanager/dto/LoginRequest.java`
- 作成：`backend/src/main/java/com/bkikichi/stockmanager/dto/AccountResponse.java`
- 作成：`backend/src/main/java/com/bkikichi/stockmanager/dto/ChangePasswordRequest.java`
- 作成：`backend/src/main/java/com/bkikichi/stockmanager/dto/MessageResponse.java`
- 作成：`backend/src/main/java/com/bkikichi/stockmanager/controller/AuthController.java`
- テスト：`backend/src/test/java/com/bkikichi/stockmanager/controller/AuthControllerTest.java`

**インターフェース：**
- 消費：タスク6の`AccountService`。
- 成果物：
  - `POST /api/auth/login` `{username, password}` → `200 {username, role, storeId, mustChangePassword}` ｜ `401 {message}`
  - `GET /api/auth/me` → `200 {username, role, storeId, mustChangePassword}` ｜ `401`
  - `POST /api/auth/logout` → `200`
  - `POST /api/auth/change-password` `{currentPassword, newPassword}` → `200 {message}` ｜ `400 {message}` ｜ `401`
  - タスク9のフロントエンド（`authApi.ts`）とタスク10の結合確認で使用する。

- [ ] **手順1：失敗するテストを書く**

`backend/src/test/java/com/bkikichi/stockmanager/controller/AuthControllerTest.java`：
```java
package com.bkikichi.stockmanager.controller;

import com.bkikichi.stockmanager.dto.ChangePasswordRequest;
import com.bkikichi.stockmanager.dto.LoginRequest;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.http.MediaType;
import org.springframework.mock.web.MockHttpSession;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;
import org.springframework.test.web.servlet.MockMvc;
import org.testcontainers.containers.PostgreSQLContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;

import static org.springframework.security.test.web.servlet.request.SecurityMockMvcRequestPostProcessors.csrf;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.post;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.jsonPath;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;

@Testcontainers
@SpringBootTest
@AutoConfigureMockMvc
class AuthControllerTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine");

    @DynamicPropertySource
    static void configureDatasource(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired
    private MockMvc mockMvc;

    @Autowired
    private ObjectMapper objectMapper;

    @Test
    void adminLoginSucceedsAndRequiresPasswordChange() throws Exception {
        mockMvc.perform(post("/api/auth/login")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(objectMapper.writeValueAsString(new LoginRequest("mg", "ChangeMe123!"))))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.role").value("ADMIN"))
                .andExpect(jsonPath("$.mustChangePassword").value(true));
    }

    @Test
    void staffLoginSucceedsAndDoesNotRequirePasswordChange() throws Exception {
        mockMvc.perform(post("/api/auth/login")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(objectMapper.writeValueAsString(new LoginRequest("staff", "staff0000"))))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.role").value("STAFF"))
                .andExpect(jsonPath("$.mustChangePassword").value(false));
    }

    @Test
    void loginWithWrongPasswordReturns401() throws Exception {
        mockMvc.perform(post("/api/auth/login")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(objectMapper.writeValueAsString(new LoginRequest("mg", "wrong-password"))))
                .andExpect(status().isUnauthorized());
    }

    @Test
    void meReturns401WhenNotAuthenticated() throws Exception {
        mockMvc.perform(get("/api/auth/me"))
                .andExpect(status().isUnauthorized());
    }

    @Test
    void meReturnsAccountAfterLogin() throws Exception {
        MockHttpSession session = new MockHttpSession();

        mockMvc.perform(post("/api/auth/login")
                        .contentType(MediaType.APPLICATION_JSON)
                        .session(session)
                        .content(objectMapper.writeValueAsString(new LoginRequest("staff", "staff0000"))))
                .andExpect(status().isOk());

        mockMvc.perform(get("/api/auth/me").session(session))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.username").value("staff"));
    }

    @Test
    void logoutInvalidatesSessionSoMeReturns401Afterward() throws Exception {
        MockHttpSession session = new MockHttpSession();

        mockMvc.perform(post("/api/auth/login")
                        .contentType(MediaType.APPLICATION_JSON)
                        .session(session)
                        .content(objectMapper.writeValueAsString(new LoginRequest("guest", "guest0000"))))
                .andExpect(status().isOk());

        mockMvc.perform(post("/api/auth/logout").session(session).with(csrf()))
                .andExpect(status().isOk());

        mockMvc.perform(get("/api/auth/me").session(session))
                .andExpect(status().isUnauthorized());
    }

    @Test
    void changePasswordSucceedsAndClearsMustChangeFlag() throws Exception {
        MockHttpSession session = new MockHttpSession();

        mockMvc.perform(post("/api/auth/login")
                        .contentType(MediaType.APPLICATION_JSON)
                        .session(session)
                        .content(objectMapper.writeValueAsString(new LoginRequest("mg", "ChangeMe123!"))))
                .andExpect(status().isOk());

        mockMvc.perform(post("/api/auth/change-password")
                        .contentType(MediaType.APPLICATION_JSON)
                        .session(session)
                        .with(csrf())
                        .content(objectMapper.writeValueAsString(
                                new ChangePasswordRequest("ChangeMe123!", "NewSecurePass456!"))))
                .andExpect(status().isOk());

        mockMvc.perform(post("/api/auth/login")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(objectMapper.writeValueAsString(new LoginRequest("mg", "NewSecurePass456!"))))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.mustChangePassword").value(false));
    }

    @Test
    void changePasswordFailsWithWrongCurrentPassword() throws Exception {
        MockHttpSession session = new MockHttpSession();

        mockMvc.perform(post("/api/auth/login")
                        .contentType(MediaType.APPLICATION_JSON)
                        .session(session)
                        .content(objectMapper.writeValueAsString(new LoginRequest("staff", "staff0000"))))
                .andExpect(status().isOk());

        mockMvc.perform(post("/api/auth/change-password")
                        .contentType(MediaType.APPLICATION_JSON)
                        .session(session)
                        .with(csrf())
                        .content(objectMapper.writeValueAsString(
                                new ChangePasswordRequest("wrong-current", "NewSecurePass456!"))))
                .andExpect(status().isBadRequest());
    }
}
```

- [ ] **手順2：テストを実行し、失敗することを確認する**

実行：`cd backend && mvn -q -Dtest=AuthControllerTest test`
期待結果：FAIL（`LoginRequest`、`ChangePasswordRequest`、`AuthController`が存在せずコンパイルエラー。また`SecurityConfig`がまだないため、タスク6で追加されたSpring Securityのデフォルト設定により`/api/auth/login`を含む全エンドポイントが認証必須になっている）

- [ ] **手順3：最小限の実装を書く**

`backend/src/main/java/com/bkikichi/stockmanager/dto/LoginRequest.java`：
```java
package com.bkikichi.stockmanager.dto;

public record LoginRequest(String username, String password) {
}
```

`backend/src/main/java/com/bkikichi/stockmanager/dto/AccountResponse.java`：
```java
package com.bkikichi.stockmanager.dto;

import com.bkikichi.stockmanager.auth.AccountRole;

public record AccountResponse(String username, AccountRole role, Long storeId, boolean mustChangePassword) {
}
```

`backend/src/main/java/com/bkikichi/stockmanager/dto/ChangePasswordRequest.java`：
```java
package com.bkikichi.stockmanager.dto;

public record ChangePasswordRequest(String currentPassword, String newPassword) {
}
```

`backend/src/main/java/com/bkikichi/stockmanager/dto/MessageResponse.java`：
```java
package com.bkikichi.stockmanager.dto;

public record MessageResponse(String message) {
}
```

`backend/src/main/java/com/bkikichi/stockmanager/auth/SecurityConfig.java`：
```java
package com.bkikichi.stockmanager.auth;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.authentication.AuthenticationManager;
import org.springframework.security.config.annotation.authentication.configuration.AuthenticationConfiguration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.web.SecurityFilterChain;
import org.springframework.security.web.csrf.CookieCsrfTokenRepository;

@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public AuthenticationManager authenticationManager(AuthenticationConfiguration config) throws Exception {
        return config.getAuthenticationManager();
    }

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            // /api/auth/loginはセッションを新規に確立するリクエストであり、
            // そのセッションに紐づくCSRFトークンがまだ存在しないため対象外にする。
            .csrf(csrf -> csrf
                .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())
                .ignoringRequestMatchers("/api/auth/login")
            )
            .sessionManagement(session -> session.sessionCreationPolicy(SessionCreationPolicy.IF_REQUIRED))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/auth/login", "/api/health").permitAll()
                .requestMatchers("/", "/index.html", "/assets/**", "/favicon.ico").permitAll()
                .anyRequest().authenticated()
            )
            .formLogin(form -> form.disable())
            .httpBasic(basic -> basic.disable());

        return http.build();
    }
}
```

`backend/src/main/java/com/bkikichi/stockmanager/controller/AuthController.java`：
```java
package com.bkikichi.stockmanager.controller;

import com.bkikichi.stockmanager.auth.Account;
import com.bkikichi.stockmanager.dto.AccountResponse;
import com.bkikichi.stockmanager.dto.ChangePasswordRequest;
import com.bkikichi.stockmanager.dto.LoginRequest;
import com.bkikichi.stockmanager.dto.MessageResponse;
import com.bkikichi.stockmanager.exception.InvalidCurrentPasswordException;
import com.bkikichi.stockmanager.exception.WeakPasswordException;
import com.bkikichi.stockmanager.service.AccountService;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.security.authentication.AuthenticationManager;
import org.springframework.security.authentication.BadCredentialsException;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.Authentication;
import org.springframework.security.core.context.SecurityContext;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.security.web.context.HttpSessionSecurityContextRepository;
import org.springframework.security.web.context.SecurityContextRepository;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("/api/auth")
public class AuthController {

    private final AuthenticationManager authenticationManager;
    private final AccountService accountService;
    private final SecurityContextRepository securityContextRepository = new HttpSessionSecurityContextRepository();

    public AuthController(AuthenticationManager authenticationManager, AccountService accountService) {
        this.authenticationManager = authenticationManager;
        this.accountService = accountService;
    }

    @PostMapping("/login")
    public ResponseEntity<?> login(@RequestBody LoginRequest request,
                                    HttpServletRequest httpRequest,
                                    HttpServletResponse httpResponse) {
        try {
            Authentication authentication = authenticationManager.authenticate(
                    new UsernamePasswordAuthenticationToken(request.username(), request.password()));

            SecurityContext context = SecurityContextHolder.createEmptyContext();
            context.setAuthentication(authentication);
            SecurityContextHolder.setContext(context);
            securityContextRepository.saveContext(context, httpRequest, httpResponse);

            Account account = accountService.findByUsername(request.username()).orElseThrow();
            return ResponseEntity.ok(toResponse(account));
        } catch (BadCredentialsException e) {
            return ResponseEntity.status(HttpStatus.UNAUTHORIZED)
                    .body(new MessageResponse("IDまたはパスワードが正しくありません"));
        }
    }

    @GetMapping("/me")
    public ResponseEntity<?> me() {
        Authentication authentication = SecurityContextHolder.getContext().getAuthentication();
        if (authentication == null || !authentication.isAuthenticated()
                || "anonymousUser".equals(authentication.getPrincipal())) {
            return ResponseEntity.status(HttpStatus.UNAUTHORIZED).build();
        }
        Account account = accountService.findByUsername(authentication.getName()).orElseThrow();
        return ResponseEntity.ok(toResponse(account));
    }

    @PostMapping("/logout")
    public ResponseEntity<?> logout(HttpServletRequest request) {
        request.getSession().invalidate();
        SecurityContextHolder.clearContext();
        return ResponseEntity.ok().build();
    }

    @PostMapping("/change-password")
    public ResponseEntity<?> changePassword(@RequestBody ChangePasswordRequest request) {
        Authentication authentication = SecurityContextHolder.getContext().getAuthentication();
        if (authentication == null || !authentication.isAuthenticated()) {
            return ResponseEntity.status(HttpStatus.UNAUTHORIZED).build();
        }

        try {
            accountService.changePassword(authentication.getName(), request.currentPassword(), request.newPassword());
            return ResponseEntity.ok(new MessageResponse("パスワードを変更しました"));
        } catch (InvalidCurrentPasswordException | WeakPasswordException e) {
            return ResponseEntity.status(HttpStatus.BAD_REQUEST).body(new MessageResponse(e.getMessage()));
        }
    }

    private AccountResponse toResponse(Account account) {
        return new AccountResponse(account.getUsername(), account.getRole(), account.getStoreId(),
                account.isMustChangePassword());
    }
}
```

- [ ] **手順4：テストを実行し、成功することを確認する**

実行：`cd backend && mvn -q -Dtest=AuthControllerTest test`
期待結果：PASS（全8件）

続けて、タスク6の補足で触れた`HealthControllerTest`（タスク2）も再実行し、`/api/health`が`SecurityConfig`の`permitAll()`設定により再び認証なしでアクセスできることを確認する：`cd backend && mvn -q -Dtest=HealthControllerTest test` → PASS

- [ ] **手順5：コミット**

```bash
git add backend/src/main/java/com/bkikichi/stockmanager/auth/SecurityConfig.java backend/src/main/java/com/bkikichi/stockmanager/dto backend/src/main/java/com/bkikichi/stockmanager/controller/AuthController.java backend/src/test/java/com/bkikichi/stockmanager/controller/AuthControllerTest.java
git commit -m "feat(backend): AuthControllerとSecurityConfigでログインAPIを追加"
```

---

### タスク8：フロントエンド雛形＋`authApi`クライアント＋ログイン画面

**ファイル：**
- 作成：`frontend/package.json`
- 作成：`frontend/vite.config.ts`
- 作成：`frontend/tsconfig.json`
- 作成：`frontend/index.html`
- 作成：`frontend/src/main.tsx`
- 作成：`frontend/src/App.tsx`
- 作成：`frontend/src/api/authApi.ts`
- 作成：`frontend/src/pages/LoginPage.tsx`
- テスト：`frontend/src/pages/LoginPage.test.tsx`
- 作成：`frontend/src/test/setup.ts`

**インターフェース：**
- 消費：タスク7の`POST /api/auth/login`、`GET /api/auth/me`、`POST /api/auth/logout`、`POST /api/auth/change-password`。
- 成果物：`login(username, password): Promise<AccountInfo>`、`fetchCurrentAccount(): Promise<AccountInfo | null>`、`logout(): Promise<void>`、`changePassword(currentPassword, newPassword): Promise<void>`、`AccountInfo { username, role: 'ADMIN'|'STAFF'|'GUEST', storeId, mustChangePassword }`。タスク9の`ChangePasswordPage`／`HomePage`が使用する。

- [ ] **手順1：`frontend/package.json`を作成する**

```json
{
  "name": "stockmanager-frontend",
  "private": true,
  "version": "0.0.1",
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "tsc -b && vite build",
    "test": "vitest run"
  },
  "dependencies": {
    "axios": "^1.7.7",
    "react": "^18.3.1",
    "react-dom": "^18.3.1",
    "react-router-dom": "^6.26.2"
  },
  "devDependencies": {
    "@testing-library/jest-dom": "^6.5.0",
    "@testing-library/react": "^16.0.1",
    "@types/react": "^18.3.5",
    "@types/react-dom": "^18.3.0",
    "@vitejs/plugin-react": "^4.3.1",
    "jsdom": "^25.0.0",
    "typescript": "^5.5.4",
    "vite": "^5.4.6",
    "vitest": "^2.1.1"
  }
}
```

- [ ] **手順2：`frontend/vite.config.ts`を作成する**

```typescript
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  server: {
    proxy: {
      '/api': 'http://localhost:8080',
    },
  },
  test: {
    environment: 'jsdom',
    globals: true,
    setupFiles: './src/test/setup.ts',
  },
});
```

- [ ] **手順3：`frontend/tsconfig.json`を作成する**

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "useDefineForClassFields": true,
    "lib": ["ES2020", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "skipLibCheck": true,
    "moduleResolution": "bundler",
    "resolveJsonModule": true,
    "isolatedModules": true,
    "noEmit": true,
    "jsx": "react-jsx",
    "strict": true,
    "types": ["vitest/globals", "@testing-library/jest-dom"]
  },
  "include": ["src"]
}
```

- [ ] **手順4：`frontend/index.html`を作成する**

```html
<!doctype html>
<html lang="ja">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>在庫管理</title>
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.tsx"></script>
  </body>
</html>
```

- [ ] **手順5：`frontend/src/test/setup.ts`を作成する**

```typescript
import '@testing-library/jest-dom';
```

- [ ] **手順6：`frontend/src/api/authApi.ts`を作成する**

```typescript
import axios from 'axios';

axios.defaults.withCredentials = true;
axios.defaults.xsrfCookieName = 'XSRF-TOKEN';
axios.defaults.xsrfHeaderName = 'X-XSRF-TOKEN';

export type AccountRole = 'ADMIN' | 'STAFF' | 'GUEST';

export interface AccountInfo {
  username: string;
  role: AccountRole;
  storeId: number;
  mustChangePassword: boolean;
}

export async function login(username: string, password: string): Promise<AccountInfo> {
  const response = await axios.post<AccountInfo>('/api/auth/login', { username, password });
  return response.data;
}

export async function fetchCurrentAccount(): Promise<AccountInfo | null> {
  try {
    const response = await axios.get<AccountInfo>('/api/auth/me');
    return response.data;
  } catch (error) {
    return null;
  }
}

export async function logout(): Promise<void> {
  await axios.post('/api/auth/logout');
}

export async function changePassword(currentPassword: string, newPassword: string): Promise<void> {
  await axios.post('/api/auth/change-password', { currentPassword, newPassword });
}
```

- [ ] **手順7：`LoginPage`の失敗するテストを書く**

`frontend/src/pages/LoginPage.test.tsx`：
```tsx
import { describe, it, expect, vi, beforeEach } from 'vitest';
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import { MemoryRouter } from 'react-router-dom';
import { LoginPage } from './LoginPage';
import * as authApi from '../api/authApi';

describe('LoginPage', () => {
  beforeEach(() => {
    vi.restoreAllMocks();
  });

  it('入力したIDとパスワードでloginを呼び出す', async () => {
    vi.spyOn(authApi, 'login').mockResolvedValue({
      username: 'staff',
      role: 'STAFF',
      storeId: 1,
      mustChangePassword: false,
    });

    render(
      <MemoryRouter>
        <LoginPage />
      </MemoryRouter>
    );

    fireEvent.change(screen.getByLabelText('ID'), { target: { value: 'staff' } });
    fireEvent.change(screen.getByLabelText('パスワード'), { target: { value: 'staff0000' } });
    fireEvent.click(screen.getByRole('button', { name: 'ログイン' }));

    await waitFor(() => expect(authApi.login).toHaveBeenCalledWith('staff', 'staff0000'));
  });

  it('ログイン失敗時にエラーメッセージを表示する', async () => {
    vi.spyOn(authApi, 'login').mockRejectedValue(new Error('unauthorized'));

    render(
      <MemoryRouter>
        <LoginPage />
      </MemoryRouter>
    );

    fireEvent.change(screen.getByLabelText('ID'), { target: { value: 'mg' } });
    fireEvent.change(screen.getByLabelText('パスワード'), { target: { value: 'wrong' } });
    fireEvent.click(screen.getByRole('button', { name: 'ログイン' }));

    expect(await screen.findByRole('alert')).toHaveTextContent('IDまたはパスワードが正しくありません');
  });
});
```

- [ ] **手順8：テストを実行し、失敗することを確認する**

実行：`cd frontend && npm install && npm test -- LoginPage`
期待結果：FAIL（`./LoginPage`モジュールが見つからない）

- [ ] **手順9：最小限の実装を書く**

`frontend/src/pages/LoginPage.tsx`：
```tsx
import { useState, FormEvent } from 'react';
import { useNavigate } from 'react-router-dom';
import { login } from '../api/authApi';

export function LoginPage() {
  const [username, setUsername] = useState('');
  const [password, setPassword] = useState('');
  const [errorMessage, setErrorMessage] = useState<string | null>(null);
  const navigate = useNavigate();

  async function handleSubmit(event: FormEvent) {
    event.preventDefault();
    setErrorMessage(null);
    try {
      const account = await login(username, password);
      if (account.role === 'ADMIN' && account.mustChangePassword) {
        navigate('/change-password');
      } else {
        navigate('/home');
      }
    } catch (error) {
      setErrorMessage('IDまたはパスワードが正しくありません');
    }
  }

  return (
    <form onSubmit={handleSubmit}>
      <h1>ログイン</h1>
      <label>
        ID
        <input value={username} onChange={(e) => setUsername(e.target.value)} />
      </label>
      <label>
        パスワード
        <input type="password" value={password} onChange={(e) => setPassword(e.target.value)} />
      </label>
      {errorMessage && <p role="alert">{errorMessage}</p>}
      <button type="submit">ログイン</button>
    </form>
  );
}
```

- [ ] **手順10：テストを実行し、成功することを確認する**

実行：`cd frontend && npm test -- LoginPage`
期待結果：PASS

- [ ] **手順11：`main.tsx`と仮の`App.tsx`を作成する**（ルーティングの最終形はタスク9で確定する）

`frontend/src/App.tsx`：
```tsx
import { BrowserRouter, Routes, Route, Navigate } from 'react-router-dom';
import { LoginPage } from './pages/LoginPage';

export function App() {
  return (
    <BrowserRouter>
      <Routes>
        <Route path="/" element={<Navigate to="/login" replace />} />
        <Route path="/login" element={<LoginPage />} />
      </Routes>
    </BrowserRouter>
  );
}
```

`frontend/src/main.tsx`：
```tsx
import React from 'react';
import ReactDOM from 'react-dom/client';
import { App } from './App';

ReactDOM.createRoot(document.getElementById('root')!).render(
  <React.StrictMode>
    <App />
  </React.StrictMode>
);
```

- [ ] **手順12：コミット**

```bash
git add frontend/package.json frontend/vite.config.ts frontend/tsconfig.json frontend/index.html frontend/src/main.tsx frontend/src/App.tsx frontend/src/api/authApi.ts frontend/src/pages/LoginPage.tsx frontend/src/pages/LoginPage.test.tsx frontend/src/test/setup.ts
git commit -m "feat(frontend): Reactアプリの雛形とログイン画面を追加"
```

---

### タスク9：パスワード変更画面・ホーム画面・ルーティング

**ファイル：**
- 作成：`frontend/src/pages/ChangePasswordPage.tsx`
- テスト：`frontend/src/pages/ChangePasswordPage.test.tsx`
- 作成：`frontend/src/pages/HomePage.tsx`
- テスト：`frontend/src/pages/HomePage.test.tsx`
- 変更：`frontend/src/App.tsx`

**インターフェース：**
- 消費：タスク8の`authApi.ts`（`changePassword`、`fetchCurrentAccount`、`AccountInfo`）。
- 成果物：`/change-password`と`/home`ルートを`App.tsx`に追加し、ログイン→（強制パスワード変更）→ホーム、という一連の流れを完成させる。タスク10の結合確認で使用する。

- [ ] **手順1：失敗するテストを書く**

`frontend/src/pages/ChangePasswordPage.test.tsx`：
```tsx
import { describe, it, expect, vi, beforeEach } from 'vitest';
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import { MemoryRouter } from 'react-router-dom';
import { ChangePasswordPage } from './ChangePasswordPage';
import * as authApi from '../api/authApi';

describe('ChangePasswordPage', () => {
  beforeEach(() => {
    vi.restoreAllMocks();
  });

  it('入力した値でchangePasswordを呼び出す', async () => {
    vi.spyOn(authApi, 'changePassword').mockResolvedValue(undefined);

    render(
      <MemoryRouter>
        <ChangePasswordPage />
      </MemoryRouter>
    );

    fireEvent.change(screen.getByLabelText('現在のパスワード'), { target: { value: 'ChangeMe123!' } });
    fireEvent.change(screen.getByLabelText('新しいパスワード'), { target: { value: 'NewPass123!' } });
    fireEvent.click(screen.getByRole('button', { name: '変更する' }));

    await waitFor(() =>
      expect(authApi.changePassword).toHaveBeenCalledWith('ChangeMe123!', 'NewPass123!')
    );
  });

  it('現在のパスワードが誤っている場合にエラーメッセージを表示する', async () => {
    vi.spyOn(authApi, 'changePassword').mockRejectedValue(new Error('bad request'));

    render(
      <MemoryRouter>
        <ChangePasswordPage />
      </MemoryRouter>
    );

    fireEvent.change(screen.getByLabelText('現在のパスワード'), { target: { value: 'wrong' } });
    fireEvent.change(screen.getByLabelText('新しいパスワード'), { target: { value: 'NewPass123!' } });
    fireEvent.click(screen.getByRole('button', { name: '変更する' }));

    expect(await screen.findByRole('alert')).toHaveTextContent(
      'パスワードの変更に失敗しました。現在のパスワードをご確認ください'
    );
  });
});
```

`frontend/src/pages/HomePage.test.tsx`：
```tsx
import { describe, it, expect, vi, beforeEach } from 'vitest';
import { render, screen } from '@testing-library/react';
import { MemoryRouter } from 'react-router-dom';
import { HomePage } from './HomePage';
import * as authApi from '../api/authApi';

describe('HomePage', () => {
  beforeEach(() => {
    vi.restoreAllMocks();
  });

  it('読み込み完了後にロールを表示する', async () => {
    vi.spyOn(authApi, 'fetchCurrentAccount').mockResolvedValue({
      username: 'staff',
      role: 'STAFF',
      storeId: 1,
      mustChangePassword: false,
    });

    render(
      <MemoryRouter>
        <HomePage />
      </MemoryRouter>
    );

    expect(await screen.findByText('ロール: STAFF')).toBeInTheDocument();
  });
});
```

- [ ] **手順2：テストを実行し、失敗することを確認する**

実行：`cd frontend && npm test -- ChangePasswordPage HomePage`
期待結果：FAIL（モジュールが見つからない）

- [ ] **手順3：最小限の実装を書く**

`frontend/src/pages/ChangePasswordPage.tsx`：
```tsx
import { useState, FormEvent } from 'react';
import { useNavigate } from 'react-router-dom';
import { changePassword } from '../api/authApi';

export function ChangePasswordPage() {
  const [currentPassword, setCurrentPassword] = useState('');
  const [newPassword, setNewPassword] = useState('');
  const [errorMessage, setErrorMessage] = useState<string | null>(null);
  const navigate = useNavigate();

  async function handleSubmit(event: FormEvent) {
    event.preventDefault();
    setErrorMessage(null);
    try {
      await changePassword(currentPassword, newPassword);
      navigate('/home');
    } catch (error) {
      setErrorMessage('パスワードの変更に失敗しました。現在のパスワードをご確認ください');
    }
  }

  return (
    <form onSubmit={handleSubmit}>
      <h1>パスワード変更（初回ログイン）</h1>
      <label>
        現在のパスワード
        <input
          type="password"
          value={currentPassword}
          onChange={(e) => setCurrentPassword(e.target.value)}
        />
      </label>
      <label>
        新しいパスワード
        <input type="password" value={newPassword} onChange={(e) => setNewPassword(e.target.value)} />
      </label>
      {errorMessage && <p role="alert">{errorMessage}</p>}
      <button type="submit">変更する</button>
    </form>
  );
}
```

`frontend/src/pages/HomePage.tsx`：
```tsx
import { useEffect, useState } from 'react';
import { useNavigate } from 'react-router-dom';
import { fetchCurrentAccount, AccountInfo } from '../api/authApi';

export function HomePage() {
  const [account, setAccount] = useState<AccountInfo | null>(null);
  const navigate = useNavigate();

  useEffect(() => {
    fetchCurrentAccount().then((result) => {
      if (result === null) {
        navigate('/login');
      } else {
        setAccount(result);
      }
    });
  }, [navigate]);

  if (!account) {
    return <p>読み込み中...</p>;
  }

  return (
    <div>
      <h1>ようこそ</h1>
      <p>ロール: {account.role}</p>
    </div>
  );
}
```

`frontend/src/App.tsx`（変更 — 新しい2ルートを追加）：
```tsx
import { BrowserRouter, Routes, Route, Navigate } from 'react-router-dom';
import { LoginPage } from './pages/LoginPage';
import { ChangePasswordPage } from './pages/ChangePasswordPage';
import { HomePage } from './pages/HomePage';

export function App() {
  return (
    <BrowserRouter>
      <Routes>
        <Route path="/" element={<Navigate to="/login" replace />} />
        <Route path="/login" element={<LoginPage />} />
        <Route path="/change-password" element={<ChangePasswordPage />} />
        <Route path="/home" element={<HomePage />} />
      </Routes>
    </BrowserRouter>
  );
}
```

- [ ] **手順4：テストを実行し、成功することを確認する**

実行：`cd frontend && npm test`
期待結果：PASS（フロントエンドの全テスト）

- [ ] **手順5：コミット**

```bash
git add frontend/src/pages/ChangePasswordPage.tsx frontend/src/pages/ChangePasswordPage.test.tsx frontend/src/pages/HomePage.tsx frontend/src/pages/HomePage.test.tsx frontend/src/App.tsx
git commit -m "feat(frontend): パスワード変更画面・ホーム画面を追加しルーティングを完成"
```

---

### タスク10：Docker Composeパッケージングとデプロイ

**ファイル：**
- 作成：`Dockerfile`
- 作成：`docker-compose.yml`
- 作成：`.env.example`

**インターフェース：**
- 消費：`backend/`（タスク2〜7）、`frontend/`（タスク8〜9）、タスク1のVPS。
- 成果物：`http://<VPS_IP>:8080`で到達可能な、ログインフローが動作するアプリ。以降のフェーズはこの成果物に画面を追加していく。

- [ ] **手順1：`Dockerfile`を作成する**

```dockerfile
# ステージ1：フロントエンドのビルド
FROM node:20-alpine AS frontend-build
WORKDIR /frontend
COPY frontend/package.json frontend/package-lock.json* ./
RUN npm install
COPY frontend/ ./
RUN npm run build

# ステージ2：バックエンドのビルド（フロントエンドのビルド成果物を静的リソースとして同梱）
FROM maven:3.9-eclipse-temurin-21 AS backend-build
WORKDIR /backend
COPY backend/pom.xml ./
RUN mvn -B dependency:go-offline
COPY backend/src ./src
COPY --from=frontend-build /frontend/dist ./src/main/resources/static
RUN mvn -B package -DskipTests

# ステージ3：実行イメージ
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY --from=backend-build /backend/target/stockmanager-0.0.1-SNAPSHOT.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

- [ ] **手順2：`docker-compose.yml`を作成する**

```yaml
services:
  postgres:
    image: postgres:16-alpine
    restart: unless-stopped
    environment:
      POSTGRES_DB: stockmanager
      POSTGRES_USER: stockmanager
      POSTGRES_PASSWORD: ${DB_PASSWORD:-stockmanager}
    volumes:
      - postgres-data:/var/lib/postgresql/data

  app:
    build:
      context: .
      dockerfile: Dockerfile
    restart: unless-stopped
    depends_on:
      - postgres
    environment:
      DB_HOST: postgres
      DB_PORT: 5432
      DB_NAME: stockmanager
      DB_USER: stockmanager
      DB_PASSWORD: ${DB_PASSWORD:-stockmanager}
    ports:
      - "8080:8080"

volumes:
  postgres-data:
```

- [ ] **手順3：`.env.example`を作成する**

```
DB_PASSWORD=changeme_in_production
```

- [ ] **手順4：ローカルでビルド・起動し、ヘルスチェックを確認する**

実行：
```bash
cp .env.example .env
docker compose up -d --build
curl http://localhost:8080/api/health
```
期待結果：`{"status":"ok"}`

- [ ] **手順5：curlでログインフローを一通り確認する**

実行：
```bash
curl -i -c /tmp/cookies.txt -X POST http://localhost:8080/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"mg","password":"ChangeMe123!"}'
```
期待結果：`HTTP/1.1 200`、`"role":"ADMIN"`と`"mustChangePassword":true`を含むJSON。

```bash
curl -i -b /tmp/cookies.txt http://localhost:8080/api/auth/me
```
期待結果：`HTTP/1.1 200`、上記と同じアカウント情報のJSON。

- [ ] **手順6（手動・人間が行う）：タスク1のConoHa VPSにデプロイする**

```bash
ssh <user>@<VPS_IP>
git clone git@github.com-bkIkichi:bk-ikichi/StockManagerApp.git
cd StockManagerApp
cp .env.example .env
# .envを編集：DB_PASSWORDをプレースホルダーではなく実際の秘密値に変更する
docker compose up -d --build
```

- [ ] **手順7：VPS外部からデプロイ済みアプリを確認する**

実行：`curl http://<VPS_IP>:8080/api/health`
期待結果：`{"status":"ok"}`

- [ ] **手順8（手動・人間が行う）：ブラウザでの動作確認**

スマホまたはPCのブラウザで`http://<VPS_IP>:8080/login`を開く。
1. `mg` / `ChangeMe123!`でログイン → パスワード変更画面に遷移することを確認
2. 新しいパスワードを入力して送信 → ホーム画面（「ロール: ADMIN」表示）に遷移することを確認
3. ログアウト（現時点ではUIのボタンなし。常設のナビゲーションができる後続フェーズで対応予定のため今回は許容）し、新しいパスワードで再ログインして変更が反映されていることを確認

- [ ] **手順9：コミット**

```bash
git add Dockerfile docker-compose.yml .env.example
git commit -m "feat: Docker ComposeでVPSにデプロイできるようパッケージング"
```

---

## フェーズ0の完了条件

- 新規VPS上で`docker compose up -d --build`を実行するだけで、手動のDBセットアップなしにスタック全体が起動する（Flyway＋初期アカウント投入で完結）
- `mg`／`staff`／`guest`それぞれでログインでき、`mg`のみ初回ログイン時にパスワード変更画面へ強制的に遷移する
- バックエンドの全テスト（`mvn test`）とフロントエンドの全テスト（`npm test`）が通る
- VPSのパブリックIPのポート8080でアプリに到達できる
