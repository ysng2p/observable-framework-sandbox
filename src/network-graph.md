---
title: Network Graph
---

# Network Graph

```js
// 使用データ読み込み
const data = await FileAttachment("data/sample.csv").csv({typed: true});

// リンクの両端ノードのID文字列を連結するのに使う文字列
const SEP = "||";
// ノード半径の最小・最大
const MinRadius = 4;
const MaxRadius = 24;
// リンク透明度の最小・最大
const MinLinkOpacity = 0.15;
const MaxLinkOpacity = 0.9;
// リンク距離の最小・最大（重いほど短くする）
const MinLinkDistance = 12;
const MaxLinkDistance = 80;

// 文字列 a, b をソートして SEP で結合して返す関数
// リンクの両端ノードのID文字列を連結するときに、順序を無視して無向グラフにするために使う
const unorderedPairKey = (a, b) => {
    // 比較を安定化するために、両方を文字列化してから大小比較する
    const A = String(a), B = String(b);
    return A < B ? `${A}${SEP}${B}` : `${B}${SEP}${A}`;
};

// リンクの source/target のID文字列を取得する関数。
// d3.forceLink(links) を使うと、source/target は
// 最初はID（文字列） → 後でノードオブジェクト に変換される。
// どちらの場合でもID文字列を使えるように、
// object なら id、それ以外（文字列のまま）ならそのまま使う。
const linkEndId = v => (typeof v === "object" ? v.id : v);

function buildGraph(data) {
    // ---------
    // リンク作成
    // ---------
    // set_id をキーとしてアイテム集合を取り出す Map を作成
    const itemsBySetId = new Map();
    for (const d of data) {
        if (!itemsBySetId.has(d.set_id)) {
            itemsBySetId.set(d.set_id, new Set());
        }
        itemsBySetId.get(d.set_id).add(d.item);
    }
    // 同じ set_id のアイテムペア（リンク対象）を作成（共起回数をカウント）
    const pairCount = new Map();
    // set_id ごとにアイテム集合を見る
    for (const itemSet of itemsBySetId.values()) {
        const itemArray = [...itemSet];
        // アイテム集合内の各アイテムペアをリンクとしてカウント
        for (let i = 0; i < itemArray.length; i++) {
            for (let j = i + 1; j < itemArray.length; j++) {
                // アイテムペアを1つの文字列にする
                const k = unorderedPairKey(itemArray[i], itemArray[j]);
                // 共起回数をカウントアップ
                pairCount.set(k, (pairCount.get(k) ?? 0) + 1);
            }
        }
    }
    // リンク配列
    const links = Array.from(pairCount, ([key, weight]) => {
        // アイテムペア文字列を分解
        const [item1, item2] = key.split(SEP);
        return {source: item1, target: item2, weight};
    })
    // ---------
    // ノード作成
    // ---------
    // 集計して結果をMap化
    const countByItem = d3.rollup(
        data,          // 集計対象データ
        v => v.length, // 各グループに対して件数（配列長）を返す
        d => d.item    // グループ化キー
    );
    const totalMeasureByItem = d3.rollup(
        data,
        v => d3.sum(v, d => d.measure),
        d => d.item
    );
    const categoriesByItem = d3.rollup(
        data,
        // 各グループに対して、カテゴリの重複なし集合を作成
        v => new Set(v.map(d => d.category)),
        d => d.item
    );
    // ノード配列
    const nodes = Array.from(
        new Set(data.map(d => d.item)),
        id => ({
            id,
            count: countByItem.get(id),
            measure: totalMeasureByItem.get(id) ?? 0,
            categories: categoriesByItem.get(id) ?? new Set(),
        })
    );
    // ---------
    // ノードをキーとして隣接ノード集合を取り出すMapを作成
    // ---------
    const neighbors = new Map(nodes.map(n => [n.id, new Set()]));
    for (const l of links) {
        const s = linkEndId(l.source), t = linkEndId(l.target);
        neighbors.get(s).add(t);
        neighbors.get(t).add(s);
    }
    // ---------
    // スケール設定
    // ---------
    // ノード半径のスケール設定
    const radiusScaleCount = d3.scaleSqrt()
        .domain(d3.extent(nodes, d => d.count)) // 入力範囲
        .range([MinRadius, MaxRadius]);         // 出力範囲
    const radiusScaleMeasure = d3.scaleSqrt()
        .domain(d3.extent(nodes, d => d.measure))
        .range([MinRadius, MaxRadius]);
    // リンク透明度のスケール設定
    const linkOpacityScale = d3.scaleLinear()
        .domain(d3.extent(links, d => d.weight))
        .range([MinLinkOpacity, MaxLinkOpacity]);
    // リンク距離のスケール設定（重いほど短くする）
    const linkDistanceScale = d3.scaleLinear()
        .domain(d3.extent(links, d => d.weight))
        .range([MaxLinkDistance, MinLinkDistance]);
    return {
        nodes,
        links,
        neighbors,
        radiusScaleCount,
        radiusScaleMeasure,
        linkOpacityScale,
        linkDistanceScale
    };
}

let {
    nodes,
    links,
    neighbors,
    radiusScaleCount,
    radiusScaleMeasure,
    linkOpacityScale,
    linkDistanceScale
} = buildGraph(data);

// width: セル幅（Observable Framework のビルトイン変数）
const SVGWidth = width; // svg 要素の幅
const SVGHeight = width * 0.475; // svg 要素の高さ
const NodeColor = "#aec7e8"; // デフォルトのノードカラー
const LinkColor = "#999"; // デフォルトのリンクカラー
const LabelFontSize = 10;
const LabelDY = -6;
const CollideRadiusMargin = 2; // ノード間の当たり判定半径の余白

// シミュレーション開始直後はグラフ領域が安定しないので少し待ってからフィットする。その待ち時間
const WaitTimeToFit = 10;

// 現在表示中ノード集合の参照（初期状態は全ノード）
let currentNodes = nodes;

// ヒストグラムの寸法
const HistWidth  = width;
const HistHeight = width * 0.1;
// 散布図の寸法
const ScatterWidth  = width;
const ScatterHeight = width * 0.35;

// ヒストグラムの描画先コンテナ
const histContainer = html`<div style="
    width:${HistWidth}px;
    height:${HistHeight}px;
    background:#f7f8fc;
"></div>`;
// 散布図の描画先コンテナ
const scatterContainer = html`<div style="
    width:${ScatterWidth}px;
    height:${ScatterHeight}px;
    background:#f7f8fc;
"></div>`;

function updateHistogram(nodeArray) {
    const values = nodeArray
        .map(d => (radiusMode === RadiusModeCount ? d.count : d.measure))
        // 数値でない値（NaN, null, undefined, Infinityなど）を除外し、ヒストグラム計算を安全にする
        .filter(v => Number.isFinite(v));

    // 描画前にクリアしておく
    histContainer.replaceChildren();

    // 空なら終了
    if (values.length === 0) return;

    // ヒストグラム生成
    const fig = Plot.plot({
        width: HistWidth,
        height: HistHeight,
        marginLeft: 64,
        marginRight: 16,
        marginBottom: 32,
        x: { label: radiusMode },
        y: { label: "ノード数（アイテム数）" },
        marks: [
            Plot.rectY(values, Plot.binX(
                // 各ビンに入ったデータの件数をカウントして y 値とする
                {y: "count"},
                {
                    x: d => d, // 値自体を x として使う
                    thresholds: 20 // ビンの数
                }
            )),
            // y=0 に水平線を引く。棒グラフの下端を明示するための基準線。
            Plot.ruleY([0])
        ]
    });

    // コンテナの中身を置き換え
    histContainer.replaceChildren(fig);
}

// 表示中ノードの count × measure を描く散布図更新関数
function updateScatter(nodeArray) {
    const data = nodeArray
        .map(d => ({
            id: d.id,
            [RadiusModeCount]: d.count,
            [RadiusModeMeasure]: d.measure
        }))
        // x,y どちらも有限数だけに限定
        .filter(d => Number.isFinite(d.count) && Number.isFinite(d.measure));

    // 描画前にクリアしておく
    scatterContainer.replaceChildren();
    // 空なら終了
    if (data.length === 0) return;

    const fig = Plot.plot({
        width: ScatterWidth,
        height: ScatterHeight,
        marginLeft: 64,
        marginRight: 16,
        marginBottom: 32,
        x: { label: RadiusModeCount },
        y: { label: RadiusModeMeasure },
        marks: [
            Plot.dot(data, {x: RadiusModeCount, y: RadiusModeMeasure}),
            // 軸の基準線
            Plot.ruleX([0]),
            Plot.ruleY([0])
        ]
    });

    // コンテナに描画
    scatterContainer.replaceChildren(fig);
}

// 何をノード半径にするか（UI上のテキストとしても使用）
const RadiusModeCount = "Count";
const RadiusModeMeasure = "Measure";
let radiusMode = RadiusModeCount;
function radiusOf(d) {
    return radiusMode === RadiusModeCount
        ? radiusScaleCount(d.count)
        : radiusScaleMeasure(d.measure);
}

const radiusModeSelector = Inputs.radio(
    // 選択肢。ラベル 兼 値になる
    [RadiusModeCount, RadiusModeMeasure],
    {
        label: "ノードサイズの基準", // UIの見出しテキスト
        value: RadiusModeCount // 初期選択値
    }
);

// イベント登録。radiusMode の変更を検知して適用
radiusModeSelector.addEventListener("change", () => {
    applyRadiusMode(radiusModeSelector.value);
});

// radiusMode の変更を適用する関数
function applyRadiusMode(nextRadiusMode) {
    radiusMode = nextRadiusMode;

    // ノード半径の変更
    nodeSel.transition().attr("r", d => radiusOf(d));

    // ノード間の当たり判定半径の変更
    sim.force("collide").radius(d => radiusOf(d) + CollideRadiusMargin);

    // ノード半径の変化による位置関係の微調整のため、力学レイアウトのシミュレーションを軽く再起動する
    sim.alpha(0.3).restart();

    updateHistogram(currentNodes);
}

// カテゴリフィルタで使う特別値
const CategoryAllKey = "__ALL__";
const CategoryUncategorizedKey = "__カテゴリなし__";
// デフォルト
const CategoryDefaultKey = CategoryAllKey;

// 重複のないカテゴリ一覧
const allCategoryNames = Array.from(new Set(data.map(d => d.category))).sort();

// セレクトの候補値（上2つは特別値、その後に実カテゴリ名を並べる）
const categoryOptions = [CategoryAllKey, CategoryUncategorizedKey, ...allCategoryNames];

// 単一選択のセレクトボックス
const categorySelector = Inputs.select(categoryOptions, {
    label: "カテゴリ",
    value: CategoryDefaultKey, // 初期値
});

// カテゴリ選択値から保持対象ノードID集合を作る
function keepIdsFromCategory(selected) {
    if (selected === CategoryAllKey) {
        return new Set(nodes.map(n => n.id));
    }
    if (selected === CategoryUncategorizedKey) {
        return new Set(nodes.filter(n => n.categories.size === 0).map(n => n.id));
    }
    return new Set(nodes.filter(n => n.categories.has(selected)).map(n => n.id));
}

// カテゴリセレクトの選択変更を検知するイベントリスナーを登録
categorySelector.addEventListener("change", () => {
    // カテゴリ選択値から保持対象ノードID集合を作る
    const keepNodeIdSet = keepIdsFromCategory(categorySelector.value);
    // サブグラフ作成
    const { subNodes, subLinks } = inducedSubgraph(nodes, links, keepNodeIdSet);
    // サブグラフを表示
    renderSubgraph(subNodes, subLinks);
});

// keepIdSet（残したいノードID集合）から、対応するサブグラフ（subNodes, subLinks）を作る関数
function inducedSubgraph(allNodes, allLinks, keepIdSet) {

    // ノード集合を絞る： 全ノードのうち、keepIdSet に含まれるノードだけを残す
    const subNodes = allNodes.filter(n => keepIdSet.has(n.id));

    // リンク集合を絞る： 両端（source, target）がどちらも keepIdSet に含まれるリンクだけを残す
    const subLinks = allLinks.filter(l => {
        const s = linkEndId(l.source), t = linkEndId(l.target);
        // 両方とも keepIdSet に含まれていれば、このリンクを残す
        return keepIdSet.has(s) && keepIdSet.has(t);
    });

    // 呼び出し側で {subNodes, subLinks} = inducedSubgraph(...) の形で受け取る
    return { subNodes, subLinks };
}

// サブグラフで描画を差し替え、力学レイアウトを再計算する関数
function renderSubgraph(subNodes, subLinks) {
    // 再バインドの前後で同じリンクを識別するための、リンクに対してキーを返す関数
    // 再バインド時、前回データは __data__ に保持されている。
    // .data(subLinks, keyOfLink)では前回データ(__data__)と
    // 今回データ(subLinks)をkeyOfLinkから得られるキーで照合する。
    const keyOfLink = l => unorderedPairKey(linkEndId(l.source), linkEndId(l.target));

    // サブグラフのリンクで線要素を差し替えるために、lineを再バインド
    linkSel = gLink.selectAll("line")
        // 前回(__data__)と今回(subLinks)を keyOfLink で得られるキーで突き合わせる
        .data(subLinks, keyOfLink)
        // 差分をDOMへ適用するために、enter/update/exit を join で反映
        .join("line")
        .attr("stroke", LinkColor)
        .attr("stroke-opacity", d => linkOpacityScale(d.weight))
        .style("pointer-events", "none"); // マウスイベントを拾わない

    // サブグラフのノードで円要素を差し替えるために、circleを再バインド
    nodeSel = gNode.selectAll("circle")
        // 前回と今回の同一ノードを d.id をキーとして結合
        .data(subNodes, d => d.id)
        .join("circle")
        .attr("r", d => radiusOf(d))
        .attr("fill", NodeColor);

    // ラベルもノードに同期させるため、text.labelを再バインド
    labelSel = gLabel.selectAll("text.label")
        .data(subNodes, d => d.id)
        // ラベルの追加・維持・削除を行うために、joinでenter/update/exitの処理を指定する
        .join("text")
        .attr("class", "label")
        .attr("font-size", LabelFontSize)
        .attr("text-anchor", "middle")
        .attr("dy", LabelDY)
        .text(d => d.id)
        .style("pointer-events", "none") // マウスイベントを拾わない
        .style("display", "none"); // 初期状態は非表示

    // ノードクリックでのサブグラフ抽出は全体グラフからサブグラフを作るので
    // サブグラフでノードクリックして新サブグラフを作る時、
    // 旧サブグラフにないが新サブグラフには存在するノードが生じる可能性がある。
    // 新規ノードにはイベントがバインドされてないので、バインドする。
    bindNodeEvents(nodeSel);

    // 対象ノードを subNodes に差し替え
    sim.nodes(subNodes);
    // 対象リンクを subLinks に差し替え
    sim.force("link").links(subLinks);
    // シミュレーションの温度 alpha をリセットし、シミュレーションをリスタート
    sim.alpha(1).restart();

    // サブグラフで画面フィット（グラフが画面外に飛ぶのを回避）
    d3.timeout(fitToContent, WaitTimeToFit);

    currentNodes = subNodes;
    updateHistogram(currentNodes);
    updateScatter(currentNodes);
}

// 隣接ノードのIDのSetを取得する関数
function neighborIdSet(nodeId) {
    // 隣接ノードがないときは空Setを返す（エラー回避）
    return neighbors.get(nodeId) || new Set();
}

// グラフ領域をSVG領域内に等比で収め、中央に配置する
function fitToContent() {
    // グラフ領域とSVG枠の間の余白（px）
    const padding = 24
    // ズーム位置合わせアニメーション時間（ms）
    const duration = 400

    // グラフ領域
    const b = g.node().getBBox();

    // コンテンツ領域（グラフ領域 + 余白）
    // 幅と高さは グラフ領域 + 両端に余白（最低でも1確保）
    const contentW = Math.max(1, b.width  + 2 * padding);
    const contentH = Math.max(1, b.height + 2 * padding);
    // 左上座標は グラフ領域の左上より余白分左上
    const contentX = b.x - padding;
    const contentY = b.y - padding;

    // 等比スケール（コンテンツ領域をSVG領域に収める倍率）
    const scale = Math.min(SVGWidth / contentW, SVGHeight / contentH);

    // 中央寄せの平行移動量
    const tx = (SVGWidth - contentW * scale) / 2 - contentX * scale;
    const ty = (SVGHeight - contentH * scale) / 2 - contentY * scale;

    // ズームtransform適用（アニメ付きで一度だけ位置合わせ）
    svg.transition()
        .duration(duration)
        .call(zoom.transform, d3.zoomIdentity.translate(tx, ty).scale(scale));
}

const tooltip = d3.select("body") // ページ全体の <body> を選択
    .append("div") // 新たに <div> 要素を追加
    .attr("class", "tooltip") // クラス名を付与（スタイル指定用）
    .style("position", "absolute") // 絶対配置にして、マウス座標に追従できるようにする
    .style("padding", "4px 8px") // 内側の余白を設定（上下4px・左右8px）
    .style("background", "rgba(0,0,0,0.7)") // 半透明の黒背景
    .style("color", "#fff") // 文字色を白に設定
    .style("border-radius", "4px") // 角を少し丸める
    .style("font-size", "12px") // 文字サイズを小さめに設定
    .style("pointer-events", "none") // ツールチップ上ではマウスイベントを無効化（ちらつき防止）
    .style("display", "none"); // 初期状態では非表示にしておく

const svg = d3.create("svg")
    .attr("width", SVGWidth)
    .attr("height", SVGHeight)
    .style("background", "#f7f8fc");

// グラフ本体
const g = svg.append("g");

// append順で表示順を制御
const gLink  = g.append("g").attr("class", "link");
const gNode  = g.append("g").attr("class", "node");
const gLabel = g.append("g").attr("class", "label");

let linkSel = gLink.selectAll("line").data(links).join("line")
    .attr("stroke", LinkColor)
    .attr("stroke-opacity", d => linkOpacityScale(d.weight))
    .style("pointer-events", "none"); // マウスイベントを拾わない

let nodeSel = gNode.selectAll("circle").data(nodes).join("circle")
    .attr("r", d => radiusOf(d))
    .attr("fill", NodeColor);

let labelSel = gLabel.selectAll("text.label").data(nodes).join("text")
    .attr("class", "label")
    .attr("font-size", LabelFontSize)
    .attr("text-anchor", "middle")
    .attr("dy", LabelDY)
    .text(d => d.id)
    .style("pointer-events", "none") // マウスイベントを拾わない
    .style("display", "none"); // 初期状態は非表示

function bindNodeEvents(nodeSel) {
    nodeSel
        // ノードにマウスホバーしたときの処理
        .on("mouseenter", (event, d) => {
            // マウスホバーしたノードの隣接ノードを取得
            const neigh = neighborIdSet(d.id);
            // 各ノードラベルについて、マウスホバーしたノード自身またはその隣接ノードである場合に、null（規定値=表示）
            labelSel.style("display", n => (n.id === d.id || neigh.has(n.id)) ? null : "none");
            // 各ノードについて、マウスホバーしたノード自身またはその隣接ノードである場合に、色変更
            nodeSel.attr("fill", n => (n.id === d.id || neigh.has(n.id)) ? "#1f77b4" : NodeColor);
            // 関連するリンクだけ色変更
            linkSel.attr("stroke", l =>
                (linkEndId(l.source) === d.id || linkEndId(l.target) === d.id)
                    ? "#FFA500"
                    : LinkColor
            );

            // ツールチップを表示状態にする
            tooltip.style("display", "block")
                // ツールチップ内容をHTMLで指定
                .html(`
                    <!-- アイテム名 -->
                    <div><strong>${d.id}</strong></div> 
                    <!-- 指標 -->
                    <div>${RadiusModeCount}: ${d.count}</div>
                    <div>${RadiusModeMeasure}: ${d.measure}</div>
                `)
                // マウス位置の右下に配置
                .style("left", `${event.pageX + 8}px`)
                .style("top", `${event.pageY + 8}px`);
        })
        .on("mousemove", (event) => {
            // マウス移動中にツールチップの位置を更新
            tooltip
                .style("left", `${event.pageX + 8}px`)
                .style("top", `${event.pageY + 8}px`);
        })
        // ノードからマウスが離れた時の処理
        .on("mouseleave", () => {
            labelSel.style("display", "none");
            nodeSel.attr("fill", NodeColor);
            linkSel
                .attr("stroke", LinkColor)
                .attr("stroke-opacity", d => linkOpacityScale(d.weight));
            tooltip.style("display", "none");
        })
        .on("click", (event, d) => {
            // クリックしたノードと隣接ノードをSetにまとめる
            const keepNodeIdSet = new Set([d.id, ...neighborIdSet(d.id)]);
            // サブグラフ作成
            const { subNodes, subLinks } = inducedSubgraph(nodes, links, keepNodeIdSet);
            // サブグラフを表示
            renderSubgraph(subNodes, subLinks);
            // カテゴリ絞り込みが解除されるため、UI上の選択カテゴリをデフォルトに戻す
            categorySelector.value = CategoryDefaultKey;
        });
    // ノード選択(selection)に対してドラッグ操作を有効化する
    nodeSel.call(
        // D3のドラッグビヘイビアを作成
        d3.drag()
            // ドラッグ開始時の処理: シミュレーションを温め、固定座標を初期化
            .on("start", (event, d) => {
                // 既にドラッグ中でなければ alphaTarget を上げてレイアウト再計算を促す
                if (!event.active) sim.alphaTarget(0.3).restart();
                // 現在位置を固定座標として設定（ドラッグ対象をつかむ）
                d.fx = d.x;
                d.fy = d.y;
            })
            // ドラッグ中の処理: マウス位置にノードを追従させる
            .on("drag", (event, d) => {
                // ドラッグ中は固定座標をマウス座標に更新
                d.fx = event.x;
                d.fy = event.y;
            })
            // ドラッグ終了時の処理: シミュレーション温度を戻し、固定を解除
            .on("end", (event, d) => {
                // これで最後のドラッグなら alphaTarget を元に戻す
                if (!event.active) sim.alphaTarget(0);
                // 固定座標を解除して力学レイアウトに再び任せる
                d.fx = null;
                d.fy = null;
            })
    );
}

bindNodeEvents(nodeSel);

// ズーム・パン操作を検知するイベントハンドラ
const zoom = d3.zoom()
    .scaleExtent([0.2, 40]) // 最小・最大ズーム倍率
    .on("zoom", (event) => {
        g.attr("transform", event.transform); // ズーム・パン操作に追従
    });
svg.call(zoom); // SVGにズーム機能を適用

// 力学レイアウトの物理シミュレーション（座標計算）
const sim = d3.forceSimulation(nodes)
    .force(
        "link",
        d3.forceLink(links)
            // links の source, target は nodes の id でノード識別する
            .id(d => d.id)
            // リンクが重いほど距離を短く（近く）する
            .distance(d => linkDistanceScale(d.weight))
    )
    // ノードを互いに反発させる
    .force("charge", d3.forceManyBody().strength(-30))
    // 全ノードの重心がSVG要素の中心に一致するように全ノードを平行移動（中央寄せ）
    .force("center", d3.forceCenter(SVGWidth / 2, SVGHeight / 2))
    // forceCollide で各ノードの「当たり判定半径」を定義
    // ノード間の距離がこの半径以下にならないよう反発させる
    // ノード半径 + 余白
    .force("collide", d3.forceCollide(d => radiusOf(d) + CollideRadiusMargin))
    // 毎フレームごとの処理
    .on("tick", () => {
        // 座標更新
        linkSel
            .attr("x1", d => d.source.x).attr("y1", d => d.source.y)
            .attr("x2", d => d.target.x).attr("y2", d => d.target.y);
        nodeSel.attr("cx", d => d.x).attr("cy", d => d.y);
        labelSel.attr("x", d => d.x).attr("y", d => d.y);
    });

// 期間範囲を取得
const dateMin = d3.min(data, d => new Date(d.date));
const dateMax = d3.max(data, d => new Date(d.date));

const startDateInput = Inputs.date({
    label: "開始日",
    value: dateMin,
    min: dateMin,
    max: dateMax
});
const endDateInput = Inputs.date({
    label: "終了日",
    value: dateMax,
    min: dateMin,
    max: dateMax
});

// 絞り込み期間の変更を適用する関数
function applyDateFilter() {
    // 絞り込み期間を取得
    const start = new Date(startDateInput.value);
    const end   = new Date(endDateInput.value);
    // 開始日・終了日が不正または範囲が逆転している場合は中断
    if (Number.isNaN(+start) || Number.isNaN(+end) || start > end) return;
    // 期間で絞り込み
    const filtered = data.filter(d => {
        const x = new Date(d.date);
        return x >= start && x <= end;
    });
    // グラフを再構築
    ({
        nodes,
        links,
        neighbors,
        radiusScaleCount,
        radiusScaleMeasure,
        linkOpacityScale,
        linkDistanceScale
    } = buildGraph(filtered));
    // 再構築したグラフを描画
    renderSubgraph(nodes, links);
    // カテゴリ絞り込みが解除されるため、UI上の選択カテゴリをデフォルトに戻す
    categorySelector.value = CategoryDefaultKey;
}

// 期間変更を監視
startDateInput.addEventListener("input", applyDateFilter);
endDateInput.addEventListener("input", applyDateFilter);

const fitButton = Inputs.button("全体フィット");
fitButton.addEventListener("click", () => fitToContent());

display(histContainer);
// display(scatterContainer);
display(svg.node());
display(radiusModeSelector);
display(categorySelector);
display(startDateInput);
display(endDateInput);
display(fitButton);

// 画面フィット
d3.timeout(fitToContent, WaitTimeToFit);
// 初期ヒストグラム描画
updateHistogram(nodes);
// 初期散布図描画
updateScatter(nodes);
```

**ノードクリック**: そのノードと隣接ノードに絞り込み（カテゴリ絞り込みは解除）  
**カテゴリ選択**: カテゴリで絞り込み（ノードクリック絞り込みは解除）  
絞り込みのたびに、期間内の全データから絞り込み直します。  
期間で絞り込むと、その他の絞り込みは解除されます。

## データ確認

使用している元データや中間データについて、確認や理解のために数件表示する。

### 元データ

このページでは、何らかのアイテム集合について、アイテムの共起関係を分析することを想定しています。
例として、マーケットバスケット分析（一緒に購入されやすい商品の分析）をイメージしたダミーデータを作成・使用しています。
1行 = ある集合（一緒に購入された商品の集合）における、あるアイテム（商品）に関するデータです。

| name       | description | example |
| ---------- | ----------- | ------- |
| `set_id`   | 集合を識別するID | 取引ID |
| `item`     | 集合内のアイテム | 商品 |
| `measure`  | その集合内でのアイテムに関する指標値 | 金額 |
| `category` | アイテムが属するカテゴリ | 商品カテゴリ |
| `date`     | 日付 | 取引日 |

```js
data.slice(0, 5)
```

### ノード

```js
nodes.slice(0, 5)
```

### リンク

```js
links.slice(0, 5)
```
