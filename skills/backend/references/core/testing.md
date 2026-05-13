# IIDP 后端测试参考

IIDP 服务测试通过 `@ExtendWith(SieEngineTestExtension.class)` 启动引擎上下文，
再用 `BaseContextHandler.getMeta()` 获取 Meta，直接调用模型服务。
这是平台提供的集成测试方式，不依赖外部 Mock 框架。

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

## 注意事项

- `@ExtendWith(SieEngineTestExtension.class)` 需要**引擎可启动**（数据库、配置就绪），不适合纯离线 CI
- 测试方法名建议格式：`{serviceName}_{场景}_{预期结果}`，如 `release_success_draftToReleased`
- 依赖引擎上下文的测试在无环境时注释掉 `@Test`，保留方法体作为手动验证脚本
- 测试文件路径：`src/test/java/com/sie/iidp/{appPkg}/{moduleName}/{ModelName}ServiceTest.java`
- 内置服务（search/find/create/update/delete）直接用 RecordSet 对应方法；
  自定义服务用 `.call("{serviceName}", args...)`

---

## TC-BE 覆盖优先级

每个 `@MethodService` 至少覆盖以下场景：

| 场景 | 测试方法命名示例 |
|---|---|
| 正常流程 | `release_success_draftToReleased` |
| 状态拒绝（非法前置状态） | `release_reject_nonDraftStatus` |
| 必填参数缺失 | `release_reject_missingId` |
| 副作用字段验证 | `release_sideEffect_actualStartTimeSet` |
