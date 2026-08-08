// ==UserScript==
// @name         生图助手
// @version      v45.6
// @description  修复ComfyUI连接CORS问题 + 独立API UI与调用链增强（v45.6）
// @author       Walkeatround & Gemini & AI Assistant
// @match        */*
// @grant        GM_xmlhttpRequest
// @connect      127.0.0.1
// @connect      localhost
// @connect      *
// ==/UserScript==

(function () {
    'use strict';
    function gmFetch(url, options = {}) {
        return new Promise((resolve, reject) => {
            GM_xmlhttpRequest({
                method: options.method || 'GET',
                url: url,
                headers: options.headers || {},
                data: options.body || undefined,
                timeout: 60000,  // 60秒超时
                onload: (response) => {
                    const res = {
                        ok: response.status >= 200 && response.status < 300,
                        status: response.status,
                        statusText: response.statusText,
                        headers: {
                            get: (name) => {
                                const header = response.responseHeaders
                                    .split('\n')
                                    .find(h => h.toLowerCase().startsWith(name.toLowerCase()));
                                return header ? header.split(': ')[1] : null;
                            }
                        },
                        text: () => Promise.resolve(response.responseText),
                        json: () => {
                            try {
                                return Promise.resolve(JSON.parse(response.responseText));
                            } catch (e) {
                                return Promise.reject(new Error('Invalid JSON: ' + response.responseText.substring(0, 100)));
                            }
                        }
                    };
                    resolve(res);
                },
                onerror: (error) => {
                    reject(new Error(`Network error: ${error.error || 'Unknown'}`));
                },
                ontimeout: () => {
                    reject(new Error('Request timeout (60s)'));
                }
            });
        });
    }

    // 智能选择：有 GM 就用 GM，没有就用普通 fetch
    const safeFetch = (typeof GM_xmlhttpRequest !== 'undefined') ? gmFetch : fetch;

    // 暴露到全局，供连接器模块使用
    window.SD_safeFetch = safeFetch;

    const SCRIPT_ID = 'sd_gen_standard_v35';
    const STORAGE_KEY = 'sd_gen_settings';
    const TEMPLATES_KEY = 'sd_gen_templates';
    const API_LOGS_KEY = 'sd_api_call_logs_v1';
    const NO_GEN_FLAG = '[no_gen]';
    const SCHEDULED_FLAG = '[scheduled]';

    // 模版编辑器当前选中的索引（移到全局避免每次打开弹窗时重置）
    let aiTplCurrentIndex = 0;
    let indepTplCurrentIndex = 0;

    const RUNTIME_LOGS = [];
    function addLog(type, msg) {
        const logLine = `[${new Date().toLocaleTimeString()}] [${type}] ${msg}`;
        RUNTIME_LOGS.push(logLine);
        console.log(logLine);
    }

    // --- 默认提示词模版 ---
    const BUILTIN_DEFAULT_TEMPLATES = {
        "默认模版": `<IMAGE_PROMPT_TEMPLATE>
You are a Visual Novel Engine. Generate story with image prompts wrapped in [IMG_GEN]...[/IMG_GEN] tags.

## 人物数据库（固定特征标签 - 必须原样复制，视为不可修改代码）
<!--人物列表-->

### 人物标签使用规则
- 严格根据剧情内容决定画哪���人物，使用对应人物的固定特征标签
- 只画剧情中实际出场的人物，不要画未出现的人物
- 提示词插入位置必须紧跟人物出场的文本段落之后，不可提前
- 人物A在前半段出场就在前半段生成，人物B在后半段出场就在后半段生成

## 核心规则
1. 每200-250字或场景/表情/动作变化时插入一个图片提示词
2. 每个提示词只描述一个角色（禁止2girls、1boy1girl等多人标签）
3. 人物数据库中的固定特征标签必须原样复制，不可修改
4. 多人互动场景：分别从每个角色的视角生成单独的提示词
5. 禁止生成URL或文件路径（如/user/images/xxx.png）

## 标签格式
`1girl/1boy, [固定特征], [表情], [服装], [姿势/动作], [视角], [环境], [光照], [质量词]`

## 姿势与动作
- 站立: standing, leaning against wall, arms crossed, hands on hips
- 坐姿: sitting, sitting on chair, sitting on bed, crossed legs, kneeling
- 躺卧: lying down, lying on back, lying on side, lying on stomach
- 动态: walking, running, jumping, reaching out, turning around
- 互动: looking at viewer, looking away, looking back, looking up, looking down
- 手部: hands together, hand on chest, hand on face, raised hand
- 特殊: crouching, bending over, stretching, hugging, embracing

## 视角与构图
- 视角: from above, from below, from side, from behind, dutch angle, pov
- 距离: close-up, upper body, cowboy shot, full body, wide shot
- 焦点: face focus, eye focus, depth of field, blurry background

## 环境背景
- 室内: bedroom, living room, classroom, office, bathroom, kitchen
- 室外: street, park, garden, beach, forest, rooftop, balcony
- 光照: sunlight, moonlight, indoor lighting, dramatic lighting, soft lighting

## 服装描述
- 上身: shirt, blouse, sweater, jacket, dress, tank top, topless
- 下身: skirt, pants, shorts, jeans, bottomless
- 足部: shoes, boots, sandals, barefoot, high heels
- 状态: wet clothes, torn clothes, disheveled clothes, naked

## 表情
smile, sad, angry, surprised, scared, blushing, gentle smile, tearful eyes, embarrassed

## 质量词后缀
highly detailed, masterpiece, best quality
</IMAGE_PROMPT_TEMPLATE>`
    };

    let DEFAULT_TEMPLATES = { ...BUILTIN_DEFAULT_TEMPLATES };
    let externalTemplatesLoaded = false;

    const OSS_BASE_URL = 'https://walkeatround.oss-cn-beijing.aliyuncs.com/src';
    const TEMPLATES_URL = `${OSS_BASE_URL}/default-templates01110441.js`;
    const CONNECTOR_URLS = {
        base: `${OSS_BASE_URL}/connectors/base-connector.js`,
        comfyui: `${OSS_BASE_URL}/connectors/comfyui-connector.js`,
        novelai: `${OSS_BASE_URL}/connectors/novelai-connector.js`,
        videoComfyui: `${OSS_BASE_URL}/connectors/video-comfyui-connector.js`,
        hfVideo: `${OSS_BASE_URL}/connectors/hf-video-connector.js`,
        defaultImageWorkflows: `${OSS_BASE_URL}/connectors/default-image-workflows.js`,
        defaultVideoWorkflows: `${OSS_BASE_URL}/connectors/default-video-workflows.js`
    };

    async function loadExternalDefaultTemplates() {
        if (typeof window.SD_DEFAULT_TEMPLATES !== 'undefined') {
            DEFAULT_TEMPLATES = { ...window.SD_DEFAULT_TEMPLATES };
            externalTemplatesLoaded = true;
            addLog('TEMPLATES', `从全局变量加载了 ${Object.keys(DEFAULT_TEMPLATES).length} 个默认模版`);
            return true;
        }

        try {
            addLog('TEMPLATES', `从 ${TEMPLATES_URL} 加载模版...`);
            const response = await safeFetch(TEMPLATES_URL);

            if (response.ok) {
                const scriptText = await response.text();
                try {
                    const evalScript = (code) => {
                        const result = eval(code);
                        return typeof SD_DEFAULT_TEMPLATES !== 'undefined' ? SD_DEFAULT_TEMPLATES : null;
                    };
                    const templates = evalScript(scriptText);

                    if (templates && typeof templates === 'object' && Object.keys(templates).length > 0) {
                        DEFAULT_TEMPLATES = { ...templates };
                        window.SD_DEFAULT_TEMPLATES = templates;
                        externalTemplatesLoaded = true;
                        addLog('TEMPLATES', `✅ 加载了 ${Object.keys(DEFAULT_TEMPLATES).length} 个默认模版`);
                        return true;
                    } else {
                        addLog('TEMPLATES', '解析模版结果为空，使用内置模版');
                    }
                } catch (evalError) {
                    addLog('TEMPLATES', `解析模版失败: ${evalError.message}，使用内置模版`);
                }
            }
        } catch (e) {
            addLog('TEMPLATES', `加载失败: ${e.message}，使用内置模版`);
        }

        return false;
    }

    const DEFAULT_SETTINGS = {
        enabled: true,
        startTag: '[IMG_GEN]',
        endTag: '[/IMG_GEN]',
        globalPrefix: 'best quality, masterpiece',
        globalSuffix: '',
        globalNegative: 'lowres, bad anatomy, bad hands, text, error, missing fingers, extra digit, fewer digits, cropped, worst quality, low quality, normal quality, jpeg artifacts, signature, waterm[...]
        promptPresets: {
            'Default': {
                prefix: 'best quality, masterpiece',
                suffix: '',
                negative: 'lowres, bad anatomy, bad hands, text, error, missing fingers, extra digit, fewer digits, cropped, worst quality, low quality, normal quality, jpeg artifacts, signature, wate[...]
            }
        },
        activePreset: 'Default',
        injectEnabled: true,
        injectDepth: 0,
        injectRole: 'system',
        selectedTemplate: "默认模版",
        characters: [
            { name: 'Character 1', tags: 'long white hair, red eyes, white dress', enabled: false }
        ],
        llmConfig: {
            baseUrl: 'https://api.deepseek.com',
            apiKey: '',
            model: 'deepseek-chat',
            maxTokens: 8192,
            temperature: 0.9,
            topP: 1.0,
            presencePenalty: 0.0,
            frequencyPenalty: 0.0
        },
        autoRefresh: false,
        autoRefreshInterval: 3000,
        generateIntervalSeconds: 1,
        autoSendGenRequest: true,
        retryCount: 3,
        retryDelaySeconds: 1,
        timeoutEnabled: false,
        timeoutSeconds: 120,
        independentApiEnabled: false,
        independentApiHistoryCount: 4,
        independentApiDebounceMs: 1000,
        independentApiRetryCount: 3,
        independentApiCustomPrompt: '',
        independentApiFilterTags: '',
        independentApiExtractTags: '',
        useDirectConnector: false,
        imageModelConfig: { activeModel: '', configs: {} },
        videoEnabled: false,
        videoUseAiPrompt: false,
        videoModelConfig: { activeModel: '', configs: {} },
        worldbookEnabled: true,
        worldbookSelections: {},
        sequentialGeneration: false,
        streamingGeneration: false,
        activePreset: '默认配置',
        apiPresets: {
            '默认配置': {
                baseUrl: 'https://api.deepseek.com',
                apiKey: '',
                model: 'deepseek-chat',
                maxTokens: 8192,
                temperature: 0.9,
                topP: 1.0,
                presencePenalty: 0.0,
                frequencyPenalty: 0.0,
                independentApiFilterTags: '',
                independentApiExtractTags: '',
                independentApiHistoryCount: 4
            }
        },
        // UI: store current selected independent model (for display)
        independentApiCurrentModel: '',
        // other templates and presets omitted for brevity (preserve existing keys)
    };

    let settings = loadSettingsFromStorage() || DEFAULT_SETTINGS;
    let customTemplates = {};
    let debounceTimer = null;
    let autoRefreshTimer = null;
    let autoRefreshPaused = false;

    function loadSettingsFromStorage() {
        try {
            const raw = localStorage.getItem(STORAGE_KEY);
            return raw ? JSON.parse(raw) : null;
        } catch (e) {
            return null;
        }
    }

    function saveSettingsToStorage() {
        try {
            localStorage.setItem(STORAGE_KEY, JSON.stringify(settings));
        } catch (e) {
            addLog('SETTINGS', `保存设置失败: ${e.message}`);
        }
    }

    // --- API 调用日志与管理 ---
    function loadApiLogs() {
        try {
            const raw = localStorage.getItem(API_LOGS_KEY);
            return raw ? JSON.parse(raw) : [];
        } catch (e) {
            return [];
        }
    }

    function saveApiLogs(logs) {
        try {
            localStorage.setItem(API_LOGS_KEY, JSON.stringify(logs.slice(0, 200))); // 限制200条
        } catch (e) {
            addLog('API_LOG', `保存日志失败: ${e.message}`);
        }
    }

    function pushApiLog(entry) {
        const logs = loadApiLogs();
        logs.unshift(entry);
        saveApiLogs(logs);
        renderAiCallLogs();
    }

    // 独立 API 请求封装：记录、重试、超时、错误码处理
    async function independentApiRequest({ url, method = 'POST', headers = {}, body = null, model = '', timeout = 120000, retryCount = 3 }) {
        const startTs = Date.now();
        const attemptLog = { timestamp: new Date().toISOString(), url, model, request: { method, headers, body }, attempts: [], final: null };

        // 增强超时处理：使用 AbortController when available
        for (let attempt = 1; attempt <= Math.max(1, retryCount); attempt++) {
            const attemptStart = Date.now();
            const controller = (typeof AbortController !== 'undefined') ? new AbortController() : null;
            const signal = controller ? controller.signal : undefined;
            let timerId = null;
            try {
                if (controller) {
                    timerId = setTimeout(() => controller.abort(), timeout);
                }
                const fetchOptions = { method, headers, body, signal };
                const fetchStart = Date.now();
                const resp = await safeFetch(url, fetchOptions);
                const tookMs = Date.now() - fetchStart;
                const text = await resp.text();
                let json = null;
                try { json = JSON.parse(text); } catch (e) { /* not json */ }

                const attemptEntry = { attempt, status: resp.status, ok: resp.ok, statusText: resp.statusText, tookMs, responseText: text.slice(0, 2000) };
                attemptLog.attempts.push(attemptEntry);

                // 记录并返回（并保存日志）
                attemptLog.final = { success: resp.ok, status: resp.status, statusText: resp.statusText, tookMs: Date.now() - startTs };
                pushApiLog(attemptLog);

                // 对于5xx错误，按策略重试特定状态
                if (!resp.ok && [502, 503, 504, 524].includes(resp.status) && attempt < retryCount) {
                    const delay = Math.pow(2, attempt) * 500; // 指数回退
                    addLog('API_RETRY', `状态 ${resp.status}，第 ${attempt} 次失败，${delay}ms 后重试`);
                    await new Promise(r => setTimeout(r, delay));
                    continue; // 下次重试
                }

                // 返回解析结果（尽量保留原接口行为）
                return { ok: resp.ok, status: resp.status, statusText: resp.statusText, text: async () => text, json: async () => json };

            } catch (err) {
                const tookMs = Date.now() - attemptStart;
                const isAbort = err && err.name === 'AbortError';
                const entry = { attempt, error: err.message || String(err), tookMs, aborted: !!isAbort };
                attemptLog.attempts.push(entry);

                // 如果是超时或者网络错误，决定是否重试
                if (attempt < retryCount) {
                    const delay = Math.pow(2, attempt) * 500;
                    addLog('API_RETRY', `网络错误或超时，第 ${attempt} 次失败: ${err.message}. ${delay}ms 后重试`);
                    await new Promise(r => setTimeout(r, delay));
                    continue;
                }

                attemptLog.final = { success: false, error: err.message, tookMs: Date.now() - startTs };
                pushApiLog(attemptLog);
                throw err;
            } finally {
                if (timerId) clearTimeout(timerId);
            }
        }
    }

    // 绑定到独立 API 使用点（示例）
    async function generateWithIndependentApi(prompt, options = {}) {
        const cfg = settings.llmConfig || settings.apiPresets && settings.apiPresets[settings.activePreset] || {};
        const baseUrl = cfg.baseUrl || settings.llmConfig.baseUrl;
        const apiKey = cfg.apiKey || settings.llmConfig.apiKey || '';
        const model = options.model || settings.independentApiCurrentModel || cfg.model || settings.llmConfig.model;

        const url = `${baseUrl}/v1/generate`;
        const headers = { 'Content-Type': 'application/json' };
        if (apiKey) headers['Authorization'] = `Bearer ${apiKey}`;
        const bodyObj = { model, prompt, max_tokens: cfg.maxTokens || 2048, temperature: cfg.temperature || 0.9 };

        // 在发送前记录到日志（用于 UI 实时显示）
        const preRecord = { time: new Date().toISOString(), userInput: prompt, worldbookInjected: settings.worldbookSelections || {}, template: settings.selectedTemplate, finalPrompt: prompt, model, requestParams: bodyObj };
        pushApiLog({ timestamp: new Date().toISOString(), note: 'pre-request', ...preRecord });

        const resp = await independentApiRequest({ url, method: 'POST', headers, body: JSON.stringify(bodyObj), model, timeout: (settings.timeoutEnabled ? settings.timeoutSeconds * 1000 : 120000), retryCount: settings.independentApiRetryCount || 3 });

        // 处理返回
        const text = await resp.text();
        let json = null;
        try { json = JSON.parse(text); } catch (e) { json = null; }

        // 记录返回
        pushApiLog({ timestamp: new Date().toISOString(), note: 'response', model, status: resp.status, statusText: resp.statusText, body: text.slice(0, 8000) });

        return { resp, text, json };
    }

    // --- UI: 独立 API 配置页面增强 ---
    function createElement(tag, props = {}, ...children) {
        const el = document.createElement(tag);
        for (const k in props) {
            if (k === 'style') Object.assign(el.style, props[k]);
            else if (k.startsWith('on') && typeof props[k] === 'function') el.addEventListener(k.substring(2).toLowerCase(), props[k]);
            else if (k === 'html') el.innerHTML = props[k];
            else el.setAttribute(k, props[k]);
        }
        for (const c of children) {
            if (typeof c === 'string') el.appendChild(document.createTextNode(c));
            else if (c instanceof Node) el.appendChild(c);
        }
        return el;
    }

    let independentApiPanel = null;

    function renderIndependentApiConfig(container) {
        if (!container) return;
        container.innerHTML = '';
        independentApiPanel = createElement('div', { class: 'sd-template-section' });

        independentApiPanel.appendChild(createElement('label', {}, '独立 API 配置'));

        // Base URL / API Key row
        const apiRow = createElement('div', { style: 'display:flex; gap:8px; align-items:center;' });
        const baseUrlInput = createElement('input', { type: 'text', value: settings.llmConfig.baseUrl || '', style: 'flex:1;', placeholder: 'Base URL (e.g. https://api.deepseek.com)' });
        baseUrlInput.addEventListener('change', () => { settings.llmConfig.baseUrl = baseUrlInput.value; saveSettingsToStorage(); });
        const apiKeyInput = createElement('input', { type: 'text', value: settings.llmConfig.apiKey || '', style: 'flex:1;', placeholder: 'API Key' });
        apiKeyInput.addEventListener('change', () => { settings.llmConfig.apiKey = apiKeyInput.value; saveSettingsToStorage(); });
        apiRow.appendChild(baseUrlInput);
        apiRow.appendChild(apiKeyInput);
        independentApiPanel.appendChild(apiRow);

        // 模型行：手动输入 / 获取按钮 / 当前实际使用模型显示
        const modelRow = createElement('div', { style: 'display:flex; gap:8px; align-items:center; margin-top:8px;' });
        const manualModelInput = createElement('input', { type: 'text', value: settings.independentApiCurrentModel || '', style: 'flex:1;', placeholder: '手动输入模型名 (如: comfy-v1)' });
        manualModelInput.addEventListener('change', () => { settings.independentApiCurrentModel = manualModelInput.value; saveSettingsToStorage(); renderIndependentApiConfig(container); });
        const fetchModelsBtn = createElement('button', { style: 'flex:0 0 130px;' }, 'API 获取模型列表');
        fetchModelsBtn.addEventListener('click', async () => {
            fetchModelsBtn.disabled = true;
            fetchModelsBtn.textContent = '获取中...';
            try {
                const cfg = settings.llmConfig || {};
                const url = (cfg.baseUrl || '').replace(/\/$/, '') + '/v1/models';
                const headers = {};
                if (cfg.apiKey) headers['Authorization'] = `Bearer ${cfg.apiKey}`;
                const resp = await independentApiRequest({ url, method: 'GET', headers, timeout: 15000, retryCount: 2 });
                const text = await resp.text();
                let list = [];
                try { const j = JSON.parse(text); list = j.data || j.models || j; } catch (e) { list = []; }
                showModelsList(list, manualModelInput);
            } catch (e) {
                alert('获取模型列表失败: ' + e.message);
            } finally {
                fetchModelsBtn.disabled = false;
                fetchModelsBtn.textContent = 'API 获取模型列表';
            }
        });
        const currentModelDisplay = createElement('div', { style: 'flex:0 0 200px; font-size:0.9em; color:#bbb;' }, `当��实际使用模型: ${settings.independentApiCurrentModel || '(未设置)'}`);
        modelRow.appendChild(manualModelInput);
        modelRow.appendChild(fetchModelsBtn);
        modelRow.appendChild(currentModelDisplay);
        independentApiPanel.appendChild(modelRow);

        // 按钮保存
        const btnRow = createElement('div', { style: 'display:flex; gap:8px; margin-top:10px;' });
        const saveBtn = createElement('button', {}, '保存设置');
        saveBtn.addEventListener('click', () => { settings.llmConfig.baseUrl = baseUrlInput.value; settings.llmConfig.apiKey = apiKeyInput.value; settings.independentApiCurrentModel = manualModelInput.value; saveSettingsToStorage(); renderIndependentApiConfig(container); alert('保存成功'); });
        btnRow.appendChild(saveBtn);
        independentApiPanel.appendChild(btnRow);

        container.appendChild(independentApiPanel);
    }

    function showModelsList(list, manualInput) {
        const dlg = createElement('div', { style: 'position:fixed; left:50%; top:50%; transform:translate(-50%,-50%); background:#111; color:#ddd; padding:16px; border-radius:8px; z-index:999999; max-height:60vh; overflow:auto; width:600px;' });
        const title = createElement('div', { style: 'font-weight:700; margin-bottom:8px;' }, '模型列表（点击使用）');
        dlg.appendChild(title);
        const ul = createElement('div', {});
        (Array.isArray(list) ? list : []).forEach(item => {
            const name = (typeof item === 'string') ? item : (item.id || item.name || JSON.stringify(item));
            const row = createElement('div', { style: 'padding:6px; border-bottom:1px solid rgba(255,255,255,0.03); cursor:pointer;' }, name);
            row.addEventListener('click', () => {
                manualInput.value = name;
                settings.independentApiCurrentModel = name;
                saveSettingsToStorage();
                document.body.removeChild(dlg);
                renderIndependentApiConfig(document.querySelector('#sd-independent-api-container'));
            });
            ul.appendChild(row);
        });
        dlg.appendChild(ul);
        const closeBtn = createElement('button', { style: 'margin-top:10px;' }, '关闭');
        closeBtn.addEventListener('click', () => document.body.removeChild(dlg));
        dlg.appendChild(closeBtn);
        document.body.appendChild(dlg);
    }

    // --- UI: 预览与调试 页面增强 ---
    function renderPreviewDebug(container) {
        if (!container) return;
        container.innerHTML = '';
        const wrap = createElement('div', { class: 'sd-template-section' });
        wrap.appendChild(createElement('label', {}, '预览与调试'));

        // 完整提示词预览
        const promptPreview = createElement('div', { style: 'background:#0f0f12; padding:12px; border-radius:8px; margin-top:8px;' });
        promptPreview.appendChild(createElement('div', { style: 'font-weight:600; margin-bottom:6px;' }, '📄 完整提示词预览'));
        const promptText = createElement('pre', { style: 'white-space:pre-wrap; max-height:200px; overflow:auto; background:transparent; color:#ddd;' }, getCompletePromptPreview());
        promptPreview.appendChild(promptText);
        wrap.appendChild(promptPreview);

        // AI 最近调用记录
        const logsSection = createElement('div', { style: 'margin-top:12px;' });
        logsSection.appendChild(createElement('div', { style: 'font-weight:600; margin-bottom:6px;' }, '🔍 AI 最近调用记录'));
        const logsContainer = createElement('div', { id: 'sd-ai-call-logs', style: 'background:#0f0f12; padding:10px; border-radius:8px; max-height:220px; overflow:auto; color:#ccc;' });
        logsSection.appendChild(logsContainer);
        wrap.appendChild(logsSection);

        container.appendChild(wrap);

        renderAiCallLogs();
    }

    function getCompletePromptPreview() {
        // 这里只生成示例：真实逻辑应基于当前消息、模板、世界书、角色等动态生成
        const prefix = settings.globalPrefix || '';
        const tmpl = settings.selectedTemplate || '默认模版';
        const main = '<!-- 插入生成的提示词 -->';
        return `${prefix}\n[Template: ${tmpl}]\n${main}`;
    }

    function renderAiCallLogs() {
        const container = document.querySelector('#sd-ai-call-logs');
        if (!container) return;
        container.innerHTML = '';
        const logs = loadApiLogs();
        if (!logs || logs.length === 0) {
            container.appendChild(createElement('div', {}, '暂无调用记录'));
            return;
        }
        // 展示最近 8 条简略记录，每条可展开
        logs.slice(0, 20).forEach((log, idx) => {
            const row = createElement('div', { style: 'border-bottom:1px solid rgba(255,255,255,0.03); padding:8px; cursor:pointer;' });
            const title = (log.note || log.timestamp || `调用 ${idx+1}`);
            row.appendChild(createElement('div', { style: 'font-weight:600; color:#fff;' }, title));
            const summary = createElement('div', { style: 'color:#aaa; font-size:12px; margin-top:6px;' }, getLogSummary(log));
            row.appendChild(summary);
            const details = createElement('pre', { style: 'display:none; white-space:pre-wrap; font-size:12px; color:#ddd; background:transparent; margin-top:8px;' }, JSON.stringify(log, null, 2));
            row.appendChild(details);
            row.addEventListener('click', () => { details.style.display = details.style.display === 'none' ? 'block' : 'none'; });
            container.appendChild(row);
        });
    }

    function getLogSummary(log) {
        if (!log) return '';
        if (log.note === 'pre-request') return `预请求 • model=${log.model || ''} • prompt(${(log.userInput||'').length} chars)`;
        if (log.note === 'response') return `返回 • status=${log.status} • model=${log.model || ''}`;
        if (log.attempts) return `请求链 • ${log.attempts.length} 次尝试 • ${log.final ? (log.final.success ? '成功' : '失败') : '未知'}`;
        return JSON.stringify(log).slice(0, 100);
    }

    // Simple integration hook: find existing config area or create a floating panel
    function injectUiEnhancements() {
        try {
            // Try to find main extension container if present
            let root = document.querySelector('#sd-root') || document.querySelector('.sd-ui-wrap') || null;
            if (!root) {
                // create a small floating button to open panel
                const floatBtn = createElement('div', { style: 'position:fixed; right:12px; bottom:12px; z-index:999999; cursor:pointer;' });
                const btn = createElement('button', { style: 'padding:8px 10px; border-radius:6px; background:#222; color:#fff; border:none;' }, '生图助手 v45.6');
                floatBtn.appendChild(btn);
                document.body.appendChild(floatBtn);
                btn.addEventListener('click', () => {
                    openControlPanel();
                });
                return;
            }

            // If root found, append panels
            const indepContainer = createElement('div', { id: 'sd-independent-api-container', style: 'margin-top:12px;' });
            root.appendChild(indepContainer);
            renderIndependentApiConfig(indepContainer);

            const previewContainer = createElement('div', { id: 'sd-preview-debug-container', style: 'margin-top:12px;' });
            root.appendChild(previewContainer);
            renderPreviewDebug(previewContainer);
        } catch (e) {
            addLog('UI', '注入 UI 时出错: ' + e.message);
        }
    }

    function openControlPanel() {
        const dlg = createElement('div', { style: 'position:fixed; left:50%; top:50%; transform:translate(-50%,-50%); background:#0e0e12; color:#ddd; padding:16px; border-radius:8px; z-index:999999; width:820px; max-height:80vh; overflow:auto;' });
        const title = createElement('div', { style: 'font-weight:700; margin-bottom:12px; font-size:16px;' }, '生图助手 v45.6 控制面板');
        dlg.appendChild(title);
        const indepContainer = createElement('div', { id: 'sd-independent-api-container' });
        dlg.appendChild(indepContainer);
        const previewContainer = createElement('div', { id: 'sd-preview-debug-container' });
        dlg.appendChild(previewContainer);
        const closeBtn = createElement('button', { style: 'margin-top:12px;' }, '关闭');
        closeBtn.addEventListener('click', () => document.body.removeChild(dlg));
        dlg.appendChild(closeBtn);
        document.body.appendChild(dlg);
        renderIndependentApiConfig(indepContainer);
        renderPreviewDebug(previewContainer);
    }

    // 初始化
    try {
        // 延迟注入，避免阻塞页面加载
        setTimeout(() => {
            injectUiEnhancements();
            loadExternalDefaultTemplates();
        }, 1500);

        // 将生成函数挂到 window，方便其他模块调用
        window.SD_generateWithIndependentApi = generateWithIndependentApi;
        window.SD_pushApiLog = pushApiLog;
        window.SD_loadApiLogs = loadApiLogs;
    } catch (e) {
        addLog('INIT', `初始化失败: ${e.message}`);
    }

})();
