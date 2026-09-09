# -2026.9-UC-
从UC网盘导出全部书签，生成标准JSON与浏览器书签HTML文件，帮助用户脱离平台锁定

## 背景

UC 网盘曾提供书签导出功能，后来悄然取消，改为只能通过 UC 客户端在用户之间批量分享，接收者仍需使用 UC App 查看，数据始终被禁锢在 UC 的生态内。这种以“方便用户”为名、行数据绑架之实的行为，令人遗憾。

本项目不破解、不攻击、不绕过任何安全机制，仅利用 UC 网盘网页版自身使用的公开接口，以自动化方式完成用户对自己数据的合法备份。

## 功能

- 获取所有书签（默认类型为“手机书签”）
- 根据 URL 去重，避免重复项
- 导出为 JSON 文件，便于程序处理
- 附带转换脚本，将 JSON 转为浏览器可直接导入的 HTML 书签文件

## 使用方法

1. 在电脑浏览器中登录 [UC 网盘](https://cloud.uc.cn)，进入书签页面。
2. 按 `F12` 打开开发者工具，切换到 **Console** 标签。
3. 复制以下脚本并粘贴运行，等待自动翻页完成后下载 JSON 文件。
4. 如需导入浏览器，再运行转换脚本，选择刚刚下载的 JSON 文件即可得到 HTML。

### 导出脚本

```javascript
// 粘贴本段代码到 Console 运行
(async () => {
  const apiUrl = 'https://cloud.uc.cn/api/bookmark/listdata';
  const getCookie = (name) => {
    const match = document.cookie.match(new RegExp('(^| )' + name + '=([^;]+)'));
    return match ? match[2] : '';
  };
  const csrfToken = getCookie('csrfToken');
  if (!csrfToken) {
    console.error('未找到 csrfToken，请确认已登录');
    return;
  }

  const allItems = [];
  const seenUrls = new Set();
  let currentPage = 1;

  while (true) {
    try {
      const res = await fetch(apiUrl, {
        method: 'POST',
        credentials: 'include',
        headers: {
          'Content-Type': 'application/json',
          'x-csrf-token': csrfToken,
          'Accept': 'application/json, text/plain, */*'
        },
        body: JSON.stringify({ cur_page: currentPage, type: 'phone', dir_guid: 0 })  //其中dir_guid可能需要修改
      });
      const json = await res.json();
      if (json.code !== 0 || !json.data || !json.data.list || json.data.list.length === 0) break;

      json.data.list.forEach(item => {
        const url = item.url || item.origin_url;
        if (url && !seenUrls.has(url)) {
          seenUrls.add(url);
          allItems.push({ title: item.title || item.name || '', url });
        }
      });

      if (!(json.data.meta && json.data.meta.has_last_page)) break;
      currentPage++;
    } catch (err) {
      console.error('请求出错，请稍后重试', err);
      return;
    }
  }

  if (allItems.length === 0) {
    console.warn('未获取到任何书签');
    return;
  }

  const blob = new Blob([JSON.stringify(allItems, null, 2)], { type: 'application/json' });
  const a = document.createElement('a');
  a.href = URL.createObjectURL(blob);
  a.download = 'uc_bookmarks_all.json';
  a.click();
  console.log(`导出成功，共 ${allItems.length} 条书签，已下载 JSON 文件`);
})();
```
注意：上述代码只能自动处理默认文件夹（根目录）中保存的书签。若存在子文件夹，需要保持在书签页面，按 F12 打开开发者工具，切换到 Network（网络）标签，在筛选框中输入`listdata`，随后点进目标子文件夹，在下面的名称中找到新出现的listdata，从 Payload（负载） 中获取 `dir_guid` 信息，用获取到的 `dir_guid` 的值替换上面代码中 `dir_guid` 的值。注意，引号也要一起复制过去，例如
```
body: JSON.stringify({ cur_page: currentPage, type: 'phone', dir_guid: "abcdexxxxxxxxx" })
```

### 转换脚本

```javascript
// 运行后选择 JSON 文件，自动下载 HTML
const fileInput = document.createElement('input');
fileInput.type = 'file';
fileInput.accept = '.json';
fileInput.onchange = async (e) => {
  const file = e.target.files[0];
  const text = await file.text();
  try {
    const items = JSON.parse(text);
    const html = `<!DOCTYPE NETSCAPE-Bookmark-file-1>
<META HTTP-EQUIV="Content-Type" CONTENT="text/html; charset=UTF-8">
<TITLE>Bookmarks</TITLE>
<H1>Bookmarks</H1>
<DL><p>
` + items.map(i => `    <DT><A HREF="${i.url}" ADD_DATE="0">${i.title || i.url}</A>`).join('\n') + `
</DL><p>`;
    const blob = new Blob([html], { type: 'text/html' });
    const a = document.createElement('a');
    a.href = URL.createObjectURL(blob);
    a.download = 'bookmarks.html';
    a.click();
    console.log(`转换完成，已下载 bookmarks.html，共 ${items.length} 条`);
  } catch (err) {
    console.error('文件解析失败，请确保选择的是由脚本导出的 JSON 文件', err);
  }
};
fileInput.click();
```

## 注意事项

- 本脚本仅用于备份个人合法拥有的书签数据，请勿用于非法用途。
- 运行脚本时，浏览器会携带你的登录 Cookie，请确保在可信环境（自己的设备、自己的账号）下操作。
- 导出的 JSON 和 HTML 文件包含你的个人网址收藏，请妥善保管，不要公开分享。

## 免责声明

本项目仅提供技术实现思路，使用者需自行承担因使用本脚本可能带来的任何风险，包括但不限于违反 UC 网盘服务条款导致的账号限制。作者不对任何后果负责。
转载请注明来源
