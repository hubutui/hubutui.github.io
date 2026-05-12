---
title: "使用 cc-switch 一站式管理 vibe coding agent 的配置"
date: 2026-05-11T00:00:00+08:00
draft: false
toc: false
images:
tags:
  - cc-switch
  - agent
  - llm
  - vibe coding
  - codex
  - gemini
  - claude code
---

## 简介

[cc-switch](https://github.com/farion1231/cc-switch) 是一款非常好用的统一管理 claude code, codex, gemini cli 的工具。

他可以管理各类 vibe coding agent 的配置，skill，mcp 等。本文主要关心的是他管理配置的功能。

## 解决的痛点

如果你在日常中使用了 claude code, codex, gemini cli 等 vibe coding 工具，还用了自己购买或者搭建的中转站的模型，就会发现给每个工具修改各自的配置文件，以及切换不同中转站的不同 api key 或者模型，会非常的繁琐且低效。而 cc-switch 可以很好的解决这个问题，并且他还支持路由模型和自动故障转移。我们可以直接添加多个供应商模型，并且全部启用，开启自动故障转移，当请求其中一个模型失败的时候会自动切换到其他的模型，免去因为某个中转站的服务不可用或者不稳定导致 vibe coding 进程不得不被中断。

## 工作原理

cc-switch 的工作原理并不复杂，他有两种工作模式：

1. 默认模式：我们给 cc-switch 指定了 claude code 等工具的配置文件路径，在我们启用 cc-switch 并指定使用某个供应商的时候，cc-switch 自动帮你讲这个供应商的配置填写到 claude code 等工具的配置文件里。
2. 路由模式：此时 cc-switch 会启动一个路由服务，我们只需要把 claude code 等工具的配置修改，请求到 cc-switch 的路由服务，由 cc-switch 决定把请求路由到哪一个供应商。这里我们一般搭配自动故障转移一起使用，此时我们添加了一组供应商服务，由 cc-switch 安装我们设置的顺序去请求对应的供应商，当某个供应商失败次数达到指定的阈值的时候，调整其优先级，确保请求到达能够正常提供服务的供应商。

## 使用建议

1. 直接添加统一供应商，一般我们使用 NewAPI，把对应的请求地址和 API key 填写上去。如果你的中转站服务商提供了不同模型的不同分组倍率，你可以创建多个 API key，每个供应商地址 + API key 都可以添加一个统一供应商，然后勾选这个是给 claude code 还是 codex 等哪个工具使用。
2. cc-switch 的路由服务默认监听的是 127.0.0.1，你也可以改成 0.0.0.0，即可供局域网内的其他设备使用，特别是方便没有 GUI 的 Linux 服务器使用。需要注意的是，目前 cc-switch 的路由服务不支持认证，请确保局域网的安全性，以免被他人恶意盗用。
3. 本机的 claude code 之类的配置文件，cc-switch 是会自动帮你修改的，但是如果是局域网内的其他机器，cc-switch 显然无法修改，因此需要你参考本机的配置文件进行修改。
4. 如果你有多个供应商 + 不同的分组 + 不同的 API key 要添加，一个个手动编辑添加也还是有点麻烦的。你可以考虑直接修改 cc-switch 的数据库。cc-switch 的数据库使用的是 sqlite3，统一供应商的配置放在 `settings` 表里的 `key` 字段为 `universal_providers` 的 `value` 字段里，是一个 json。而一般的供应商则放在 `providers` 表里。根据其表结构信息，即可实现使用脚本导入。

以下是一个 python 脚本，用于导入 universal_providers 和 providers。这个脚本的 universal_providers 导入的是 NewAPI 这种统一供应商，providers 则是用于导入 modelscope 的免费模型。建议只导入 universal_providers 即可，modelscope 提供的免费模型其实调用次数有限，无法满足编程需要的。

```python
#!/usr/bin/env python3
#
# 通过读写数据库来导入 provider 和 universal provider
# provider 直接写入 providers 表
# universal provider 写入 settings 表里的 key 字段为 universal_providers 的 value
# universal provider 的 id 根据 f"{base_url}:{api_key}:{app}:{model}" 来生成固定的 uuid5，确保重复插入不会出错
# provider 的 id 根据 f"{base_url}:{api_key}:{model}"
#
import uuid
from enum import Enum
from typing import Literal
import argparse
import json
import time
from pathlib import Path
from urllib.parse import urlparse

from loguru import logger
from sqlalchemy import Boolean, Column, Integer, String, create_engine
from sqlalchemy.orm import Session, declarative_base

from pydantic import BaseModel, ConfigDict, Field


class _CamelAliasModel(BaseModel):
    """基类：所有带 alias 的子模型统一支持 snake_case 和 camelCase 两种传值方式。"""

    model_config = ConfigDict(populate_by_name=True)


class CustomEndpoint(_CamelAliasModel):
    url: str
    added_at: int = Field(alias="addedAt")
    last_used: int | None = Field(default=None, alias="lastUsed")


class UsageScript(_CamelAliasModel):
    enabled: bool
    language: str
    code: str
    timeout: int | None = None
    api_key: str | None = Field(default=None, alias="apiKey")
    base_url: str | None = Field(default=None, alias="baseUrl")
    access_token: str | None = Field(default=None, alias="accessToken")
    user_id: str | None = Field(default=None, alias="userId")
    template_type: str | None = Field(default=None, alias="templateType")
    auto_query_interval: int | None = Field(default=None, alias="autoQueryInterval")


class ProviderTestConfig(_CamelAliasModel):
    enabled: bool = False
    test_model: str | None = Field(default=None, alias="testModel")
    timeout_secs: int | None = Field(default=None, alias="timeoutSecs")
    test_prompt: str | None = Field(default=None, alias="testPrompt")
    degraded_threshold_ms: int | None = Field(default=None, alias="degradedThresholdMs")
    max_retries: int | None = Field(default=None, alias="maxRetries")


class AuthBindingSource(str, Enum):
    PROVIDER_CONFIG = "provider_config"
    MANAGED_ACCOUNT = "managed_account"


class AuthBinding(_CamelAliasModel):
    source: AuthBindingSource = AuthBindingSource.PROVIDER_CONFIG
    auth_provider: str | None = Field(default=None, alias="authProvider")
    account_id: str | None = Field(default=None, alias="accountId")


class ProviderMeta(_CamelAliasModel):
    custom_endpoints: dict[str, CustomEndpoint] = Field(default_factory=dict)
    common_config_enabled: bool | None = Field(
        default=None, alias="commonConfigEnabled"
    )
    usage_script: UsageScript | None = None
    endpoint_auto_select: bool | None = Field(default=None, alias="endpointAutoSelect")
    is_partner: bool | None = Field(default=None, alias="isPartner")
    partner_promotion_key: str | None = Field(default=None, alias="partnerPromotionKey")
    cost_multiplier: str | None = Field(default=None, alias="costMultiplier")
    pricing_model_source: str | None = Field(default=None, alias="pricingModelSource")
    limit_daily_usd: str | None = Field(default=None, alias="limitDailyUsd")
    limit_monthly_usd: str | None = Field(default=None, alias="limitMonthlyUsd")
    test_config: ProviderTestConfig | None = Field(default=None, alias="testConfig")
    api_format: (
        Literal["anthropic", "openai_chat", "openai_responses", "gemini_native"] | None
    ) = Field(default=None, alias="apiFormat")
    auth_binding: AuthBinding | None = Field(default=None, alias="authBinding")
    api_key_field: Literal["ANTHROPIC_AUTH_TOKEN", "ANTHROPIC_API_KEY"] | None = Field(
        default=None, alias="apiKeyField"
    )
    is_full_url: bool | None = Field(default=None, alias="isFullUrl")
    prompt_cache_key: str | None = Field(default=None, alias="promptCacheKey")
    codex_fast_mode: bool | None = Field(default=None, alias="codexFastMode")
    provider_type: str | None = Field(default=None, alias="providerType")
    github_account_id: str | None = Field(default=None, alias="githubAccountId")


class UniversalProviderApps(BaseModel):
    claude: bool = False
    codex: bool = False
    gemini: bool = False


class ClaudeModelConfig(_CamelAliasModel):
    model: str | None = None
    haiku_model: str | None = Field(default=None, alias="haikuModel")
    sonnet_model: str | None = Field(default=None, alias="sonnetModel")
    opus_model: str | None = Field(default=None, alias="opusModel")


class CodexModelConfig(_CamelAliasModel):
    model: str | None = None
    reasoning_effort: str | None = Field(default=None, alias="reasoningEffort")


class GeminiModelConfig(BaseModel):
    model: str | None = None


class UniversalProviderModels(BaseModel):
    claude: ClaudeModelConfig | None = None
    codex: CodexModelConfig | None = None
    gemini: GeminiModelConfig | None = None


class UniversalProvider(_CamelAliasModel):
    """统一供应商，跨应用共享配置。"""

    id: str
    name: str
    provider_type: str = Field(alias="providerType")
    apps: UniversalProviderApps
    base_url: str = Field(alias="baseUrl")
    api_key: str = Field(alias="apiKey")
    models: UniversalProviderModels = Field(default_factory=UniversalProviderModels)
    website_url: str | None = Field(default=None, alias="websiteUrl")
    notes: str | None = None
    icon: str | None = None
    icon_color: str | None = Field(default=None, alias="iconColor")
    meta: ProviderMeta | None = None
    created_at: int | None = Field(default=None, alias="createdAt")
    sort_index: int | None = Field(default=None, alias="sortIndex")


# settings 表中存储的顶层结构：provider id -> UniversalProvider
UniversalProvidersMap = dict[str, UniversalProvider]


class ClaudeEnvConfig(BaseModel):
    ANTHROPIC_BASE_URL: str
    ANTHROPIC_AUTH_TOKEN: str
    ANTHROPIC_MODEL: str
    ANTHROPIC_DEFAULT_HAIKU_MODEL: str
    ANTHROPIC_DEFAULT_SONNET_MODEL: str
    ANTHROPIC_DEFAULT_OPUS_MODEL: str


class ClaudeSettingsConfig(BaseModel):
    env: ClaudeEnvConfig


class Provider(_CamelAliasModel):
    """供应商，对应 providers 表。"""

    id: str
    name: str
    settings_config: ClaudeSettingsConfig = Field(alias="settingsConfig")
    website_url: str | None = Field(default=None, alias="websiteUrl")
    category: str | None = None
    created_at: int | None = Field(default=None, alias="createdAt")
    sort_index: int | None = Field(default=None, alias="sortIndex")
    notes: str | None = None
    meta: ProviderMeta | None = None
    icon: str | None = None
    icon_color: str | None = Field(default=None, alias="iconColor")
    in_failover_queue: bool = Field(default=False, alias="inFailoverQueue")


CLAUDE_DEFAULT_MODELS = ClaudeModelConfig(
    model="claude-opus-4-6",
    haiku_model="claude-haiku-4-6",
    sonnet_model="claude-sonnet-4-6",
    opus_model="claude-opus-4-6",
)

CODEX_DEFAULT_MODELS = CodexModelConfig(
    model="gpt-5.4",
    reasoning_effort="high",
)


def create_provider(
    *,
    base_url: str,
    api_key: str,
    model: str,
    name: str,
    website_url: str | None = None,
    icon: str | None = None,
) -> Provider:
    """创建单个 Provider（Claude app type）。"""
    seed = f"{base_url}:{api_key}:{model}"
    provider_id = str(uuid.uuid5(uuid.NAMESPACE_URL, seed))
    settings_config = ClaudeSettingsConfig(
        env=ClaudeEnvConfig(
            ANTHROPIC_BASE_URL=base_url,
            ANTHROPIC_AUTH_TOKEN=api_key,
            ANTHROPIC_MODEL=model,
            ANTHROPIC_DEFAULT_HAIKU_MODEL=model,
            ANTHROPIC_DEFAULT_SONNET_MODEL=model,
            ANTHROPIC_DEFAULT_OPUS_MODEL=model,
        )
    )
    return Provider(
        id=provider_id,
        name=name,
        settings_config=settings_config,
        website_url=website_url,
        icon=icon,
    )


def create_universal_provider(
    *,
    base_url: str,
    api_key: str,
    name: str,
    provider_type: str = "newapi",
    app: Literal["claude", "codex"] = "claude",
    model: str | None = None,
) -> UniversalProvider:
    """创建单个 UniversalProvider。"""
    seed = f"{base_url}:{api_key}:{app}"
    if model:
        seed = f"{seed}:{model}"
    provider_id = str(uuid.uuid5(uuid.NAMESPACE_URL, seed))
    apps_config = UniversalProviderApps(**{app: True})
    if app == "codex":
        codex_models = (
            CODEX_DEFAULT_MODELS.model_copy(update={"model": model})
            if model
            else CODEX_DEFAULT_MODELS
        )
        models = UniversalProviderModels(codex=codex_models)
    else:
        claude_models = (
            CLAUDE_DEFAULT_MODELS.model_copy(
                update={
                    "model": model,
                    "haiku_model": model,
                    "sonnet_model": model,
                    "opus_model": model,
                }
            )
            if model
            else CLAUDE_DEFAULT_MODELS
        )
        models = UniversalProviderModels(claude=claude_models)
    return UniversalProvider(
        id=provider_id,
        name=name,
        provider_type=provider_type,
        apps=apps_config,
        base_url=base_url,
        api_key=api_key,
        models=models,
        icon="newapi",
    )


if __name__ == "__main__":

    Base = declarative_base()

    class Settings(Base):
        __tablename__ = "settings"
        key = Column(String, primary_key=True)
        value = Column(String)

    class ProviderRow(Base):
        __tablename__ = "providers"
        id = Column(String, primary_key=True)
        app_type = Column(String, primary_key=True)
        name = Column(String, nullable=False)
        settings_config = Column(String, nullable=False)
        website_url = Column(String)
        category = Column(String)
        created_at = Column(Integer)
        sort_index = Column(Integer)
        notes = Column(String)
        icon = Column(String)
        icon_color = Column(String)
        meta = Column(String, nullable=False, default="{}")
        is_current = Column(Boolean, nullable=False, default=False)
        in_failover_queue = Column(Boolean, nullable=False, default=False)

    MODELSCOPE_BASE_URL = "https://api-inference.modelscope.cn"
    MODELSCOPE_WEBSITE = "https://modelscope.cn"
    DB_PATH = Path.home() / ".cc-switch" / "cc-switch.db"

    parser = argparse.ArgumentParser(description="批量生成 providers 并写入数据库")
    parser.add_argument("--api-key", required=True, help="API key")
    parser.add_argument(
        "--target",
        choices=["universal", "providers"],
        required=True,
        help="写入目标：universal 写 settings 表，providers 写 providers 表",
    )
    parser.add_argument(
        "--db", default=str(DB_PATH), help=f"数据库路径（默认 {DB_PATH}）"
    )
    parser.add_argument(
        "--dry-run", action="store_true", help="只打印结果，不写入数据库"
    )

    universal_group = parser.add_argument_group("universal provider 选项")
    universal_group.add_argument(
        "--base-url", help="API base URL（--target=universal 时必填）"
    )
    universal_group.add_argument(
        "--name-suffix",
        default=None,
        help="名称后缀，最终名称为 域名 + suffix（如 --name-suffix '1x'）",
    )
    universal_group.add_argument(
        "--universal-provider-type",
        default="newapi",
        help="供应商类型（默认 newapi）",
    )
    universal_group.add_argument(
        "--app",
        choices=["claude", "codex"],
        default="claude",
        help="目标应用（默认 claude）",
    )
    universal_group.add_argument(
        "--model",
        default=None,
        help="覆盖默认模型，claude 时同时设置 model/haiku/sonnet/opus，codex 时设置 model",
    )

    providers_group = parser.add_argument_group("providers 表选项")
    providers_group.add_argument(
        "--models-file",
        help="模型列表文件，每行一个模型名，# 开头或空行忽略（--target=providers 时必填）",
    )

    args = parser.parse_args()

    engine = create_engine(f"sqlite:///{args.db}")

    if args.target == "universal":
        if not args.base_url:
            parser.error("--base-url is required when --target=universal")

        hostname = urlparse(args.base_url).hostname or args.base_url
        name = f"{hostname} - {args.name_suffix}" if args.name_suffix else hostname

        up = create_universal_provider(
            base_url=args.base_url,
            api_key=args.api_key,
            name=name,
            provider_type=args.universal_provider_type,
            app=args.app,
            model=args.model,
        )
        new_data = {up.id: up.model_dump(by_alias=True, exclude_none=True)}

        with Session(engine) as session:
            row = session.get(Settings, "universal_providers")
            existing: dict = json.loads(row.value) if row and row.value else {}
            existing.update(new_data)
            merged_json = json.dumps(existing, ensure_ascii=False)

            if args.dry_run:
                logger.info(json.dumps(existing, ensure_ascii=False, indent=2))
            else:
                if row:
                    row.value = merged_json
                else:
                    session.add(Settings(key="universal_providers", value=merged_json))
                session.commit()
                logger.info(
                    f"已写入 settings.universal_providers，共 {len(existing)} 个 provider"
                )
    else:
        if not args.models_file:
            parser.error("--models-file is required when --target=providers")

        with open(args.models_file, encoding="UTF-8") as f:
            models = [
                line.strip()
                for line in f
                if line.strip() and not line.strip().startswith("#")
            ]

        base_url = args.base_url or MODELSCOPE_BASE_URL
        now_ms = int(time.time() * 1000)

        with Session(engine) as session:
            inserted = 0
            updated = 0
            for i, model_name in enumerate(models):
                provider = create_provider(
                    base_url=base_url,
                    api_key=args.api_key,
                    model=model_name,
                    name=f"ModelScope - {model_name}",
                    website_url=MODELSCOPE_WEBSITE,
                    icon="modelscope",
                )
                settings_config_json = json.dumps(
                    provider.model_dump(by_alias=True, include={"settings_config"})[
                        "settingsConfig"
                    ],
                    ensure_ascii=False,
                )

                existing_row = session.get(ProviderRow, (provider.id, "claude"))

                if args.dry_run:
                    action = "UPDATE" if existing_row else "INSERT"
                    logger.info(
                        f"  [{action}] {provider.id}  {provider.name} {settings_config_json}"
                    )
                    continue

                if existing_row:
                    existing_row.name = provider.name
                    existing_row.settings_config = settings_config_json
                    existing_row.website_url = provider.website_url
                    existing_row.icon = provider.icon
                    existing_row.in_failover_queue = True
                    updated += 1
                else:
                    session.add(
                        ProviderRow(
                            id=provider.id,
                            app_type="claude",
                            name=provider.name,
                            settings_config=settings_config_json,
                            website_url=provider.website_url,
                            created_at=now_ms + i,
                            meta="{}",
                            icon=provider.icon,
                            in_failover_queue=True,
                        )
                    )
                    inserted += 1

            if args.dry_run:
                logger.info(f"共 {len(models)} 条（dry-run，未写入）")
            else:
                session.commit()
                logger.info(f"已写入 providers 表：插入 {inserted}，更新 {updated}")
```

## 总结

本文简单介绍了 cc-switch 如何一站式管理 claude code, codex 等 vibe coding 工具的配置，并支持添加多个供应商，支持自动故障转移等。
