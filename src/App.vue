<script setup>
import { ref } from 'vue'
import * as kdbxweb from 'kdbxweb'

// Reactive state
const files = ref([])
const tree = ref([])
const password = ref('')
const showPassword = ref(false)
const status = ref('')

// Utility functions
function parseChromeBookmarksHtml(text) {
    // Simple HTML parser for Chrome's exported bookmarks.html
    const parser = new DOMParser();
    const doc = parser.parseFromString(text, 'text/html');
    function walkDl(dl) {
        const items = [];
        for (const node of dl.childNodes) {
            if (node.nodeName === 'DT') {
                const h3 = node.querySelector('h3');
                const a = node.querySelector('a');
                if (h3) {
                    const subDl = node.querySelector('dl');
                    items.push({
                        type: 'folder',
                        title: h3.textContent || 'Folder',
                        children: subDl ? walkDl(subDl) : []
                    });
                } else if (a) {
                    items.push({ type: 'bookmark', title: a.textContent || a.getAttribute('href'), url: a.getAttribute('href') });
                }
            }
        }
        return items;
    }
    const topDl = doc.querySelector('dl');
    if (!topDl) return [];
    return walkDl(topDl);
}

function parseChromeBookmarksJson(text) {
    const obj = JSON.parse(text);
    function nodeToItems(node) {
        const items = [];
        if (!node) return items;
        if (node.type === 'url') return [{ type: 'bookmark', title: node.name || node.url, url: node.url }];
        if (node.children && Array.isArray(node.children)) {
            for (const ch of node.children) {
                if (ch.type === 'folder') {
                    items.push({ type: 'folder', title: ch.name || 'Folder', children: nodeToItems({ children: ch.children }) });
                } else if (ch.type === 'url') {
                    items.push({ type: 'bookmark', title: ch.name || ch.url, url: ch.url });
                }
            }
        }
        return items;
    }
    // Chrome's bookmark JSON often has roots: bookmark_bar, other, synced
    const roots = obj.roots || obj;
    const res = [];
    for (const k of Object.keys(roots)) {
        if (roots[k] && roots[k].children) {
            res.push({ type: 'folder', title: roots[k].name || k, children: nodeToItems(roots[k]) });
        }
    }
    return res;
}

function mergeTrees(trees) {
    // trees: array of item arrays. Need to merge preserving folders, and dedupe URLs (first occurrence)
    const seen = new Set();
    function cloneFilter(items) {
        const out = [];
        for (const it of items) {
            if (it.type === 'bookmark') {
                if (!it.url) continue;
                if (seen.has(it.url)) continue;
                seen.add(it.url);
                out.push({ ...it });
            } else if (it.type === 'folder') {
                const children = cloneFilter(it.children || []);
                out.push({ type: 'folder', title: it.title, children });
            }
        }
        return out;
    }
    // Flatten by creating a top-level container with combined children
    const combined = [];
    for (const t of trees) {
        for (const it of t) combined.push(it);
    }
    return cloneFilter(combined);
}

function escapeHtml(s) {
    return (s || '').toString().replace(/[&<>"]/g, (c) => ({ '&': '&amp;', '<': '&lt;', '>': '&gt;', '"': '&quot;' }[c] || c))
}

// Event handlers
async function handleFiles(ev) {
    const list = Array.from(ev.target.files || ev.files || [])
    files.value = list
    status.value = 'Parsing files...'
    const trees = []
    for (const f of list) {
        const txt = await f.text()
        // try json first
        let parsed = []
        try {
            parsed = parseChromeBookmarksJson(txt)
        } catch (e) {
            try {
                parsed = parseChromeBookmarksHtml(txt)
            } catch (e2) {
                console.warn('failed parse', e2)
            }
        }
        trees.push(parsed)
    }
    tree.value = mergeTrees(trees)
    status.value = '解析完成'
}

function renderTree(items) {
    return items.map((it) => {
        if (it.type === 'folder') {
            return (`<div class="folder"><div><strong>${escapeHtml(it.title)}</strong></div><div class="tree">${renderTree(it.children || []).join('')}</div></div>`)
        }
        return (`<div class="entry">${escapeHtml(it.title)} — <span class="small">${escapeHtml(it.url)}</span></div>`)
    })
}

async function generateKdbx() {
    if (!tree.value.length) { status.value = '没有书签数据'; return }
    status.value = '生成 KDBX 中...'
    const cred = new kdbxweb.Credentials(kdbxweb.ProtectedValue.fromString(password.value || ''))
    const db = kdbxweb.Kdbx.create(cred, 'Bookmarks')

    // Force AES KDF and KDBX v3 to avoid Argon2 NotImplemented error
    try {
        db.setKdf(kdbxweb.Consts.KdfId.Aes)
    } catch (e) {
        // ignore if not supported
    }
    try {
        db.setVersion(3)
    } catch (e) {
        // ignore if not supported
    }

    const defaultGroup = db.getDefaultGroup()

    function addItemsToGroup(items, parentGroup) {
        for (const it of items) {
            if (it.type === 'folder') {
                const g = db.createGroup(parentGroup, it.title || 'Folder')
                addItemsToGroup(it.children || [], g)
            } else if (it.type === 'bookmark') {
                const e = db.createEntry(parentGroup)
                e.fields.set('Title', it.title || '')
                e.fields.set('URL', it.url || '')
            }
        }
    }

    addItemsToGroup(tree.value, defaultGroup)

    let data
    try {
        data = await db.save()
    } catch (e) {
        // If Argon2 not implemented or other KDF issues, try forcing AES + v3 and save again
        const isArgon2Error =
            (e && e.message && /argon2/i.test(e.message)) ||
            (e && e.code === kdbxweb.Consts.ErrorCodes.NotImplemented)
        if (isArgon2Error) {
            try {
                db.setKdf(kdbxweb.Consts.KdfId.Aes)
                db.setVersion(3)
                data = await db.save()
            } catch (e2) {
                status.value = `保存失败：${e2.message || e2}`
                throw e2
            }
        } else {
            status.value = `保存失败：${e.message || e}`
            throw e
        }
    }
    const blob = new Blob([data], { type: 'application/octet-stream' })
    const url = URL.createObjectURL(blob)
    const a = document.createElement('a')
    a.href = url
    const pad = (n) => String(n).padStart(2, '0')
    const now = new Date()
    const filename = `书签_${now.getFullYear()}${pad(now.getMonth() + 1)}${pad(now.getDate())}_${pad(now.getHours())}${pad(now.getMinutes())}${pad(now.getSeconds())}.kdbx`
    a.download = filename
    a.click()
    URL.revokeObjectURL(url)
    status.value = `生成完成，文件 ${filename} 已下载`
}
</script>

<template>
    <div class="container">
        <div class="header">
            <div>
                <div class="title">Bookmarks → KeePassXC (.kdbx)</div>
                <div class="notice">所有处理在本地浏览器完成，不会上传到服务器</div>
            </div>
        </div>

        <div class="card">
            <div class="controls">
                <label class="file-input btn">
                    选择书签文件
                    <input type="file" multiple @change="handleFiles" accept=".html,application/json,text/json" />
                </label>
                <button class="btn primary" @click="generateKdbx">生成并下载 .kdbx</button>
                <div class="password">
                    <input :type="showPassword ? 'text' : 'password'" class="input" placeholder="设置 KDBX 密码（可选）"
                        v-model="password" />
                    <button class="btn eye-btn" @click="showPassword = !showPassword"
                        :title="showPassword ? '隐藏密码' : '显示密码'">
                        {{ showPassword ? '👁️' : '👁️‍🗨️' }}
                    </button>
                </div>
            </div>
            <div class="small" style="margin-top:8px">状态：{{ status }}</div>
        </div>

        <div class="card">
            <div><strong>解析预览</strong></div>
            <div v-html="renderTree(tree)"></div>
        </div>

        <div class="footer">
            <div class="small">提示：仅保留首次出现的 URL，保留目录结构。</div>
        </div>
    </div>
</template>
