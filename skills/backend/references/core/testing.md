# IIDP 后端测试参考

两种测试策略分层使用，互相补充：

| 策略 | 工具 | 需要数据库 | 适合测试 |
|---|---|---|---|
| **集成测试** | `SieEngineTestExtension` | ✅ 需要 | 完整服务流程、真实数据写入验证 |
| **单元测试** | Mockito | ❌ 不需要 | 业务逻辑判断、状态校验、异常分支 |

> H2 内存数据库：IIDP 引擎**不支持** H2 方言，不可用。

---

## 基本结构

```java
package com.sie.iidp.{appPkg}.{moduleName};

import com.sie.snest.engine.container.Meta;
import com.sie.snest.engine.context.BaseContextHandler;
import com.sie.snest.engine.rule.Filter;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import server.annotation.SieEngineTestExtension;

import static org.junit.jupiter.api.Assertions.*;

@ExtendWith(SieEngineTestExtension.class)
class {ModelName}ServiceTest {

    @Test
    void {testMethodName}() {
        Meta meta = BaseContextHandler.getMeta();
        assertNotNull(meta);

        // 调用目标服务
        Object result = meta.get("{model_name}").call("{serviceName}", /* args... */);
        assertNotNull(result);
    }
}
```

关键点：
- `@ExtendWith(SieEngineTestExtension.class)` 放在**类级别**，测试前启动引擎
- `BaseContextHandler.getMeta()` 获取当前请求级 Meta 上下文
- `meta.get("{model_name}")` 获取目标模型的 RecordSet
- `.call("{serviceName}", args...)` 调用 `@MethodService(name="...")`

---

## 状态机服务测试（最常用）

```java
@ExtendWith(SieEngineTestExtension.class)
class WorkOrderServiceTest {

    @Test
    void release_success_draftToReleased() {
        Meta meta = BaseContextHandler.getMeta();

        // 前置：确认记录存在且状态为 DRAFT
        Object record = meta.get("{model_name}").call("find",
            Filter.equal("id", testId), null, null, null, null);
        assertNotNull(record);

        // 执行状态变更服务
        Object result = meta.get("{model_name}").call("release", testId);
        assertNotNull(result);

        // 验证后置状态
        Object updated = meta.get("{model_name}").call("find",
            Filter.equal("id", testId), null, null, null, null);
        // 断言 status == RELEASED
    }

    @Test
    void release_reject_nonDraftStatus() {
        Meta meta = BaseContextHandler.getMeta();
        // 前置：记录状态为 RELEASED（非 DRAFT）
        // 预期：抛出 ModelException
        assertThrows(Exception.class, () ->
            meta.get("{model_name}").call("release", releasedId)
        );
    }
}
```

---

## 指定租户上下文

需要在特定租户下运行时，用 `new Meta(tenantId, context)` 创建隔离上下文：

```java
@Test
void testWithTenantContext() {
    Meta outerMeta = BaseContextHandler.getMeta();
    assertNotNull(outerMeta);

    try (Meta meta = new Meta("{tenantId}", outerMeta.getContext())) {
        Object result = meta.get("{model_name}").search(
            new Filter(), null, null, null, null);
        assertNotNull(result);
    } catch (Exception e) {
        fail("测试失败：" + e.getMessage());
    }
}
```

---

## 异步服务测试

```java
@Test
void testAsync() {
    Meta meta = BaseContextHandler.getMeta();
    assertNotNull(meta);

    Object result = meta.get("{model_name}").executeAsync(
        () -> Meta.getCurrentMeta().get("{model_name}"),
        "search"
    );
    assertNotNull(result);
}
```

---

## Mockito 单元测试（无需数据库）

`BaseContextHandler.getMeta()` 是静态方法，需要 `mockito-inline` 依赖支持静态 mock。

### POM 依赖

```xml
<dependency>
    <groupId>org.mockito</groupId>
    <artifactId>mockito-inline</artifactId>
    <version>4.x.x</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.mockito</groupId>
    <artifactId>mockito-junit-jupiter</artifactId>
    <version>4.x.x</version>
    <scope>test</scope>
</dependency>
```

> `mockito-inline` 包含 `mockito-core`，不重复引入。Java 8 使用 Mockito 4.x；Java 11+ 可用 5.x。

### 状态机服务单元测试

```java
import org.mockito.MockedStatic;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
class {ModelName}ServiceUnitTest {

    @Mock
    private Meta meta;

    @Mock
    private RecordSet recordSet;

    @Test
    void {serviceName}_reject_illegalStatus() {
        try (MockedStatic<BaseContextHandler> ctx =
                 mockStatic(BaseContextHandler.class)) {

            ctx.when(BaseContextHandler::getMeta).thenReturn(meta);
            when(meta.get("{model_name}")).thenReturn(recordSet);

            // 模拟查库返回非法前置状态的记录
            Map<String, Object> record = Map.of("id", 1L, "status", "RELEASED");
            when(recordSet.call(eq("find"), any(), any(), any(), any(), any()))
                .thenReturn(record);

            // 预期抛出 ModelException
            assertThrows(ModelException.class, () ->
                meta.get("{model_name}").call("{serviceName}", 1L)
            );
        }
    }

    @Test
    void {serviceName}_success_sideEffectFieldSet() {
        try (MockedStatic<BaseContextHandler> ctx =
                 mockStatic(BaseContextHandler.class)) {

            ctx.when(BaseContextHandler::getMeta).thenReturn(meta);
            when(meta.get("{model_name}")).thenReturn(recordSet);

            // 模拟查库返回合法前置状态
            Map<String, Object> record = Map.of("id", 1L, "status", "DRAFT");
            when(recordSet.call(eq("find"), any(), any(), any(), any(), any()))
                .thenReturn(record);

            // 执行服务
            meta.get("{model_name}").call("{serviceName}", 1L);

            // 验证副作用字段被写入（验证 update 被调用且包含目标字段）
            verify(recordSet).call(eq("update"), argThat(args ->
                args.toString().contains("status") &&
                args.toString().contains("RELEASED")
            ));
        }
    }
}
```

### 选择哪种测试

| 要测的内容 | 用哪种 |
|---|---|
| 状态拒绝逻辑（非法状态 → ModelException） | Mockito 单元测试 |
| 必填参数校验（缺 id → ValidationException） | Mockito 单元测试 |
| 副作用字段是否被赋值 | Mockito 单元测试 |
| 数据真实写入数据库 | SieEngineTestExtension 集成测试 |
| 跨模型 RPC 调用链路 | SieEngineTestExtension 集成测试 |

---

## 集成测试注意事项

- `@ExtendWith(SieEngineTestExtension.class)` 需要**引擎可启动**（数据库、配置就绪），不适合纯离线 CI
- 无法启动引擎时注释掉 `@Test`，保留方法体作为手动验证脚本
- 测试方法名格式：`{serviceName}_{场景}_{预期结果}`，如 `release_success_draftToReleased`
- 测试文件路径：`src/test/java/com/sie/iidp/{appPkg}/{moduleName}/{ModelName}ServiceTest.java`
- 内置服务直接用 RecordSet 对应方法；自定义服务用 `.call("{serviceName}", args...)`

---

## TC-BE 覆盖优先级

每个 `@MethodService` 至少覆盖以下场景：

| 场景 | 测试方法命名示例 |
|---|---|
| 正常流程 | `release_success_draftToReleased` |
| 状态拒绝（非法前置状态） | `release_reject_nonDraftStatus` |
| 必填参数缺失 | `release_reject_missingId` |
| 副作用字段验证 | `release_sideEffect_actualStartTimeSet` |
