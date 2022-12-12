---
title: "浏览器前进/后退缓存机制探索"
date: "2022-12-12"
tags: ["browser", "programming"]
---

[浏览器的前进/后退缓存](https://web.dev/bfcache/)（Back/forward cache, 简称 bfcache）指的是浏览器会缓存浏览过的页面，在点击前进或后退按钮（或 JS 调用 history API）时不请求服务器而直接从缓存中加载静态文件的机制，这种缓存机制可以大幅提高浏览器前进/后退时的速度，优化用户体验。然而，浏览器主动进行的缓存也可能给前端开发带来一些困扰，尤其是缓存机制在各个浏览器上的实现不尽相同时就更是如此。因此总结不同主流浏览器具体的缓存机制是有必要的。

<!-- excerpt -->

## 测试环境

测试使用的机型为 MacBook Air (M1, 2020)，系统版本为 macOS 12.6 (21G115)。浏览器分别为 Safari(16.0)，Chrome(108.0)，Edge(108.0)，Firefox(106.0)。PHP 版本为 8.2 使用 PHP 内置测试服务器进行测试。

使用的代码如下：
```PHP
// index.php
<a href="/add.php">Go To Add Page</a>
<div>
    <lable>PHP</lable>
    <ul>
        <?php
            $cache_file = 'cache.json';
            $cache = json_decode(file_get_contents($cache_file), true);
            foreach ($cache['data'] as $item) {
                echo "<li>{$item}</li>";
            }
        ?>
    </ul>
</div>
<div>
    <lable>JavaScript</lable>
    <ul id="js-list"></ul>
</div>
<script>
    document.addEventListener("DOMContentLoaded", () => {
        const loadJsList = init => {
            fetch("/cache.php", init)
                .then(res => res.json())
                .then(result => {
                    document.querySelector("#js-list").innerHTML = result.data.map(
                        item => `<li>${item}</li>`
                    ).join("");
                });
            }

        const onAdd = () => {
            event.preventDefault();

            loadJsList({
                method: "POST",
                headers: {
                    "Content-Type": "application/json"
                },
                body: JSON.stringify({
                    newDate: document.querySelector("input[name='new-data']").value
                })
            });

                return false;
            }

            loadJsList();
        });
</script>

// add.php
<input type="text" name="new-data">
<button onclick="onAdd()">ADD</button>
<script>
    const onAdd = () => {
        event.preventDefault();

        const newDate = document.querySelector("input[name='new-data']").value;
        fetch("/cache.php", {
            method: "POST",
            headers: {
                "Content-Type": "application/json"
            },
            body: JSON.stringify({
                newDate
            })
        })
        .then(res => res.ok ? history.go(-1) : alert(res.statusText));

        return false;
    }
</script>

// cache.php
<?php

$cache_file = 'cache.json';
$cache = file_get_contents($cache_file);

if ($_SERVER['REQUEST_METHOD'] == 'POST') {
    $params = json_decode(file_get_contents('php://input'), true);
    $newDate = $params['newDate'];

    $cache = json_decode($cache, true);
    $cache['data'][] = $newDate;

    $cache = json_encode($cache);
    file_put_contents($cache_file, $cache);
}

echo $cache;
```

测试时从 index 页面前往 add 页面进行添加操作，操作成功后通过 `history.go(-1)`（和浏览器的回退按钮行为一致）返回 index。index 页面分别在服务器和浏览器上加载并渲染数据列表，因此可以通过观察 index 页面上数据加载的情况来推测浏览器的缓存机制。

## 测试结果

Safari 测试结果如下，在 add 页面添加数据 safari 后返回 index 页面，PHP 和 JS 加载的数据都没有更新。

![safari add data](/assets/img/bfcache/safari-add.png "safari 添加数据")

![safari result](/assets/img/bfcache/safari-result.png "safari 测试结果")

Chrome 测试结果如下，在 add 页面添加数据 chrome 后返回 index 页面，PHP 和 JS 加载的数据都没有更新。

![chrome add data](/assets/img/bfcache/chrome-add.png "chrome 添加数据")

![chrome result](/assets/img/bfcache/chrome-result.png "chrome 测试结果")

Edge 测试结果如下，在 add 页面添加数据 edge 后返回 index 页面，PHP 加载的数据都没有更新，而 JS 加载的数据进行了更新。

![edge add](/assets/img/bfcache/edge-add.png "edge 添加数据")

![edge result](/assets/img/bfcache/edge-result.png "edge 测试结果")

Firefox 测试结果如下，在 add 页面添加数据 firefox 后返回 index 页面，PHP 和 JS 加载的数据都没有更新。

![firefox add](/assets/img/bfcache/firefox-add.png "firefox 添加数据")

![firefox result](/assets/img/bfcache/firefox-result.png "firefox 测试结果")

从上述结果可以看到，由于浏览器后退时从缓存中加载了页面，因此四个浏览器中使用 PHP 加载的数据都没有更新，而只有 Edge 浏览器在后退时重新执行了页面的 JS 代码，因此 JS 加载的数据进行了更新。此外在其他浏览器中添加了数据，在 Edge 浏览器中进行前进后退操作（没有与服务器进行通信）时 JS 代码不会执行，因此推测 Edge 只有在页面与服务器通信之后执行前进后退操作才会执行 JS 代码。

## 禁用缓存的方法

1. 通过在服务器的响应头中添加 `cache-control:no-store` 的 header 可以提示浏览器不缓存相关内容，但是该 header 并不是强制的。
```PHP
<?php
    // do not store cache
    header('cache-control:no-store');
    $cache_file = 'cache.json';
    $cache = json_decode(file_get_contents($cache_file), true);
    foreach ($cache['data'] as $item) {
        echo "<li>{$item}</li>";
    }
?>
```

添加 header 后 Chrome 可以正常更新数据。

![chrome header result](/assets/img/bfcache/chrome-header-result.png "chrome 测试结果")

但是 Safari 依然无法更新数据。

![safari header result](/assets/img/bfcache/safari-header-result.png "safari 测试结果")

2. 监听[页面转换事件](https://developer.mozilla.org/zh-CN/docs/Web/API/PageTransitionEvent)：pageshow

当网页在加载完成或卸载后会触发页面传输事件，而事件的参数 persisted 将标记页面是否从缓存（Backforward Cache）中加载，因此通过监听 pageshow 事件可以在页面从缓存中加载时进行强制刷新。

```javaScript
window.addEventListener("pageshow", (event) => {
    if (event.persisted) {
        window.location.reload();
    }
});
```

添加上述代码后，Safari 在进行前进后退操作时可以正常更新数据。

![safari refresh result](/assets/img/bfcache/safari-refresh-result.png "safari 测试结果")

## 不足

上面的测试只覆盖了 Mac 上比较常见的几种浏览器，在不同的操作系统和移动端浏览器可能有不同的行为，不同浏览器的不同版本对页面转换事件的支持程度也有区别。此外，缓存机制还有可能和数据的大小以及缓存时间有关，这些都需要进一步的测试。
