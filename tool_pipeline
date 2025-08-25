# -*- coding: utf-8 -*-
"""
Tool-First + Built-in RAG (RU Law) for Open WebUI.
Приоритет: инструменты -> встроенный RAG -> ответ.
Цитирование [id] только при наличии атрибута id у <source>.
"""

import re
import json
import orjson
from typing import Any, Dict

# --- Триггеры для инструментов (подгоняй при желании) ---
TRIGGERS = {
    "Garant_document_check": [
        "проверь реквизит", "проверь документ", "статус документа",
        "дата последнего изменения", "срок действия", "территори", "зарегистрир"
    ],
    "Garant_single_search": [
        "найди документ", "точное название", "конкретный документ", "где текст закона"
    ],
    "Garant_multisearch": [
        "подборка документов", "список документов", "топ-10", "подходящие документы", "поиск по гарант"
    ],
    "Enhanced Web Scrape": [
        "что на этой странице", "проанализируй ссылку", "веб-страниц", "скачай страницу", "html", "url", "ссылка http", "https://", "http://"
    ],
    "Markdown → PDF Server": [
        "сделай pdf", "pdf из markdown", "в pdf", "сохранить в pdf", "экспорт в pdf"
    ],
}

# Жёсткие правила приоритета, если совпали сразу несколько
TOOL_PRIORITY = [
    "Markdown → PDF Server",
    "Enhanced Web Scrape",
    "Garant_document_check",
    "Garant_single_search",
    "Garant_multisearch",
]

SYSTEM_GUARD = (
    "Ты — ассистент с инструментами. ПРИОРИТЕТ: 1) ИНСТРУМЕНТЫ → 2) встроенный RAG → 3) ответ.\n"
    "Сначала прими JSON-решение:\n"
    '{"need_tool": true|false, "tool": "Enhanced Web Scrape|Garant_single_search|Garant_multisearch|Garant_document_check|Markdown → PDF Server|none", "why": "1-2 причины"}\n'
    "Если need_tool=true — СНАЧАЛА вызови указанный инструмент (через function call), дождись результата и только затем пиши ответ. "
    "После инструмента при необходимости используй встроенный RAG для уточнений и цитат.\n\n"
    "Правила RAG-ответа по законам РФ:\n"
    "- Опирайся на извлечённые из RAG фрагменты. Если редакции конфликтуют — укажи даты и выбери актуальную.\n"
    "- Ссылки вида [id] добавляй ТОЛЬКО когда в соответствующем источнике есть явный атрибут id у <source>. "
    "Если id нет — ссылку не ставь.\n"
    "- Формат: Краткий вывод → Ключевые нормы → Примечания/исключения → Дисклеймер «Не является юридической консультацией».\n"
)

ANSWER_TEMPLATE = (
    "### Task:\n"
    "Ответь на пользовательский запрос, используя при наличии встроенный RAG-контекст (Open WebUI его добавит автоматически). "
    "Ставь ссылки [id] ТОЛЬКО если у источника есть <source id=\"...\">.\n\n"
    "### Guidelines:\n"
    "- Отвечай на русском.\n"
    "- Если данных из контекста мало — перечисли, чего не хватает.\n"
    "- Не используй теги в ответе; только текст и, при наличии id, квадратные ссылки.\n\n"
    "### Output:\n"
    "Краткий вывод: <1–3 предложения>\n\n"
    "Ключевые нормы:\n"
    "- <Акт, статья/часть/пункт — краткое пояснение>. [id — только если есть]\n"
    "- <...>\n\n"
    "Примечания и исключения:\n"
    "- <уточнения по процедурам, срокам, редакциям>\n\n"
    "Если данных не хватает:\n"
    "- <какие уточнения нужны>\n\n"
    "Не является юридической консультацией."
)

def pick_tool_by_trigger(user_text: str) -> Dict[str, Any]:
    text = user_text.lower()
    fired = []
    for tool, keys in TRIGGERS.items():
        for k in keys:
            if k in text:
                fired.append(tool)
                break
    if not fired:
        return {"need_tool": False, "tool": "none", "why": "no trigger"}
    # приоритет
    for t in TOOL_PRIORITY:
        if t in fired:
            return {"need_tool": True, "tool": t, "why": "trigger"}
    return {"need_tool": True, "tool": fired[0], "why": "trigger"}

class Pipeline:
    async def on_start(self, ctx: Dict[str, Any]) -> Dict[str, Any]:
        return {"system_prompt": SYSTEM_GUARD}

    async def on_message(self, prompt: str, ctx: Dict[str, Any]) -> Dict[str, Any]:
        router_hint = pick_tool_by_trigger(prompt)
        # передадим «подсказку» модели отдельным сообщением system, а также напомним про шаблон вывода
        return {
            "system_prompt": SYSTEM_GUARD,
            "prepend_messages": [
                {"role": "system", "content": "Router hint (rule-based): " + json.dumps(router_hint, ensure_ascii=False)},
                {"role": "system", "content": ANSWER_TEMPLATE},
            ],
            # важное: не подменяем сам prompt; встроенный RAG продолжит работать как обычно
            "override_user_prompt": None,
        }
