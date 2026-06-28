<?php
// public/products.php
require_once __DIR__ . '/../src/Database.php';
require_once __DIR__ . '/../src/Auth.php';
require_once __DIR__ . '/../src/Helpers.php';

Auth::startSession();
$pdo = Database::get();
Auth::tryRememberLogin($pdo);
if (!Auth::check()) {
    header('Location: /login.php');
    exit;
}

/**
 * GET parameters and defaults
 */
$perPageOptions = [10,20,50,100,200,'all'];
$per_page_raw = isset($_GET['per_page']) ? $_GET['per_page'] : '50';
$per_page = ($per_page_raw === 'all') ? 'all' : (int)$per_page_raw;
if ($per_page !== 'all' && !in_array($per_page, [10,20,50,100,200], true)) $per_page = 50;

$page = isset($_GET['page']) ? max(1, (int)$_GET['page']) : 1;

$allowedSorts = [
        'id' => 'p.id',
        'name' => 'p.name',
        'supplier' => 's.name',
        'brand' => 'b.name',
        'group' => 'g.name',
        'price_base' => 'pr.price_base',
        'price_after' => 'pr.price_after_discount'
];
$sort = isset($_GET['sort']) && array_key_exists($_GET['sort'], $allowedSorts) ? $_GET['sort'] : 'price_after';
$order = (isset($_GET['order']) && strtolower($_GET['order']) === 'desc') ? 'DESC' : 'ASC';

/**
 * Filters: supplier, brand, material
 */
$rawSuppliers = $_GET['supplier'] ?? [];
$rawBrands = $_GET['brand'] ?? [];
$rawMaterials = $_GET['material'] ?? [];

if (!is_array($rawSuppliers)) $rawSuppliers = [$rawSuppliers];
if (!is_array($rawBrands)) $rawBrands = [$rawBrands];
if (!is_array($rawMaterials)) $rawMaterials = [$rawMaterials];

$useAllSupplier = in_array('all', $rawSuppliers, true) || empty($rawSuppliers);
$useAllBrand = in_array('all', $rawBrands, true) || empty($rawBrands);
$useAllMaterial = in_array('all', $rawMaterials, true) || empty($rawMaterials);

$suppliers = $useAllSupplier ? [] : array_values(array_filter(array_map('intval', $rawSuppliers)));
$brands = $useAllBrand ? [] : array_values(array_filter(array_map('intval', $rawBrands)));
$materials = $useAllMaterial ? [] : array_values(array_filter(array_map('intval', $rawMaterials)));

$tagSearch = trim($_GET['tags'] ?? '');

/**
 * Helper: build query string preserving filters and sort for links
 */
function buildQuery(array $overrides = []) {
    $base = $_GET;
    foreach ($overrides as $k => $v) {
        if ($v === null) unset($base[$k]);
        else $base[$k] = $v;
    }
    return http_build_query($base);
}

/**
 * Prepare supplier/brand/material lists for selects and maps
 */
$supList = $pdo->query('SELECT id, name FROM suppliers WHERE is_active = 1 ORDER BY name')->fetchAll(PDO::FETCH_ASSOC);
$brandList = $pdo->query('SELECT id, name FROM brands WHERE is_active = 1 ORDER BY name')->fetchAll(PDO::FETCH_ASSOC);
$materialList = $pdo->query('SELECT id, name FROM materials ORDER BY name')->fetchAll(PDO::FETCH_ASSOC);
$groupRows = $pdo->query('SELECT id, name FROM groups_catalog ORDER BY name')->fetchAll(PDO::FETCH_ASSOC);
$groupMap = [];
foreach ($groupRows as $g) $groupMap[$g['id']] = $g['name'];

/**
 * Fetch results
 */
$results = [];
$totalItems = 0;

if ($tagSearch !== '') {
    $tokens = preg_split('/\s+/u', $tagSearch);
    $allMatches = searchProductsByTags($pdo, $tokens, array_filter([
            'supplier_id' => $suppliers ?: null,
            'brand_id' => $brands ?: null,
            'material_id' => $materials ?: null
    ]), 10000, 0);

    $supplierNames = [];
    $brandNames = [];
    foreach ($supList as $s) $supplierNames[$s['id']] = $s['name'];
    foreach ($brandList as $b) $brandNames[$b['id']] = $b['name'];

    usort($allMatches, function($a, $b) use ($sort, $order, $supplierNames, $brandNames, $groupMap) {
        $dir = ($order === 'ASC') ? 1 : -1;
        switch ($sort) {
            case 'name':
                return $dir * strcmp($a['name'] ?? '', $b['name'] ?? '');
            case 'supplier':
                $sa = $supplierNames[$a['supplier_id']] ?? '';
                $sb = $supplierNames[$b['supplier_id']] ?? '';
                return $dir * strcmp($sa, $sb);
            case 'brand':
                $ba = $brandNames[$a['brand_id']] ?? '';
                $bb = $brandNames[$b['brand_id']] ?? '';
                return $dir * strcmp($ba, $bb);
            case 'group':
                $ga = $groupMap[$a['group_id']] ?? '';
                $gb = $groupMap[$b['group_id']] ?? '';
                return $dir * strcmp($ga, $gb);
            case 'price_base':
                return $dir * (floatval($a['price_base'] ?? 0) <=> floatval($b['price_base'] ?? 0));
            case 'price_after':
                $a_price_after_discount = $a['price_after_second_discount'] ?? $a['price_after_discount'] ?? 0;
                $b_price_after_discount = $b['price_after_second_discount'] ?? $b['price_after_discount'] ?? 0;
                return $dir * (floatval($a_price_after_discount) <=> floatval($b_price_after_discount));
            case 'id':
            default:
                return $dir * (intval($a['id']) <=> intval($b['id']));
        }
    });

    $totalItems = count($allMatches);

    if ($per_page === 'all') {
        $results = $allMatches;
        $totalPages = 1;
        $page = 1;
    } else {
        $totalPages = max(1, (int)ceil($totalItems / $per_page));
        if ($page > $totalPages) $page = $totalPages;
        $offset = ($page - 1) * $per_page;
        $results = array_slice($allMatches, $offset, $per_page);
    }
} else {
	$sort = isset($_GET['sort']) && array_key_exists($_GET['sort'], $allowedSorts) ? $_GET['sort'] : 'id';
	$order = (isset($_GET['order']) && strtolower($_GET['order']) === 'asc') ? 'ASC' : 'DESC';

    $sql = "
      SELECT 
        p.id, 
        p.name, 
        p.supplier_id, 
        p.brand_id, 
        p.group_id, 
        p.konkav,
        p.konvex,
        p.konvex_konkav,
        pr.price_base, 
        pr.price_after_discount,
		pr.date_effective,
		pr.second_discount_percent,
		pr.price_after_second_discount
      FROM products p
      LEFT JOIN (
        SELECT t1.product_id, t1.price_base, t1.price_after_discount, t1.date_effective, t1.second_discount_percent,t1.price_after_second_discount
        FROM prices t1
        INNER JOIN (
          SELECT product_id, MAX(date_effective) AS max_date
          FROM prices
          GROUP BY product_id
        ) t2 ON t1.product_id = t2.product_id AND t1.date_effective = t2.max_date
      ) pr ON pr.product_id = p.id
      LEFT JOIN suppliers s ON s.id = p.supplier_id
      LEFT JOIN brands b ON b.id = p.brand_id
      LEFT JOIN groups_catalog g ON g.id = p.group_id
      WHERE 1=1
    ";
    $params = [];

    if (!empty($suppliers)) {
        $in = implode(',', array_fill(0, count($suppliers), '?'));
        $sql .= " AND p.supplier_id IN ($in)";
        foreach ($suppliers as $v) $params[] = $v;
    }
    if (!empty($brands)) {
        $in = implode(',', array_fill(0, count($brands), '?'));
        $sql .= " AND p.brand_id IN ($in)";
        foreach ($brands as $v) $params[] = $v;
    }
    if (!empty($materials)) {
        $in = implode(',', array_fill(0, count($materials), '?'));
        $sql .= " AND p.material_id IN ($in)";
        foreach ($materials as $v) $params[] = $v;
    }

    // Count total
    $countSql = "SELECT COUNT(*) FROM (" . $sql . ") tcount";
    $countStmt = $pdo->prepare($countSql);
    $countStmt->execute($params);
    $totalItems = (int)$countStmt->fetchColumn();

    // Append ORDER BY and LIMIT/OFFSET (unless per_page == 'all')
    $orderBy = $allowedSorts[$sort] . ' ' . $order;
    $sql .= " ORDER BY $orderBy";

    if ($per_page === 'all') {
        // no LIMIT/OFFSET
        $stmt = $pdo->prepare($sql);
        $stmt->execute($params);
        $results = $stmt->fetchAll(PDO::FETCH_ASSOC);
        $totalPages = 1;
        $page = 1;
    } else {
        $sql .= " LIMIT ? OFFSET ?";
        $params[] = $per_page;
        $params[] = ($page - 1) * $per_page;
        $stmt = $pdo->prepare($sql);
        $stmt->execute($params);
        $results = $stmt->fetchAll(PDO::FETCH_ASSOC);
        $totalPages = max(1, (int)ceil($totalItems / $per_page));
        if ($page > $totalPages) $page = $totalPages;
    }
}

/**
 * sort link helper
 */
function sortLink($columnKey, $label, $currentSort, $currentOrder) {
    $nextOrder = 'desc';
    if ($currentSort === $columnKey && strtolower($currentOrder) === 'desc') $nextOrder = 'asc';
    $qs = buildQuery(['sort' => $columnKey, 'order' => $nextOrder, 'page' => 1]);
    $arrow = '';
    if ($currentSort === $columnKey) {
        $arrow = strtolower($currentOrder) === 'asc' ? ' ▲' : ' ▼';
    }
    return "<a href=\"?{$qs}\" class=\"text-decoration-none\">{$label}{$arrow}</a>";
}

/**
 * Utility: fetch name map
 */
function fetchNameMap(PDO $pdo, $table) {
    $rows = $pdo->query("SELECT id, name FROM {$table}")->fetchAll(PDO::FETCH_ASSOC);
    $map = [];
    foreach ($rows as $r) $map[$r['id']] = $r['name'];
    return $map;
}
$supplierMap = fetchNameMap($pdo, 'suppliers');
$brandMap = fetchNameMap($pdo, 'brands');
$materialMap = fetchNameMap($pdo, 'materials');

?>
<!doctype html>
<html lang="fa" dir="rtl">
<head>
    <meta charset="utf-8">
    <title>لیست کالاها - NobelOptic</title>
    <meta name="viewport" content="width=device-width,initial-scale=1">
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.2/dist/css/bootstrap.rtl.min.css" rel="stylesheet">
    <! -- link href="/css/bootstrap.rtl.min.css" rel="stylesheet" -->
    <style>
        table.table td, table.table th { text-align: right; vertical-align: middle; }
        .edit-btn { display:inline-flex; align-items:center; gap:6px; padding:6px 10px; border-radius:6px; border:1px solid #d0d0d0; background:#f5f5f5; color:#333; text-decoration:none; }
        .edit-btn svg { display:block; }
        .per-page-select { min-width:60px; max-width:90px; font-size:14px; }
        .filter-note { font-size:12px; color:#666; }
        /* toolbar: force LTR layout so left/right positions are stable regardless of page RTL */
        .toolbar { display:flex; justify-content:space-between; align-items:center; gap:12px; margin-bottom:8px; direction:ltr; }
        .toolbar .left, .toolbar .right { direction:rtl; } /* keep inner text RTL */
        @media (max-width:767px){ .toolbar { flex-direction:column; align-items:stretch; } .toolbar .left, .toolbar .right { direction:rtl; } }
        .pagination { margin:0; }
    </style>
</head>
<body class="bg-white">
<nav class="navbar navbar-light bg-light">
    <div class="container">
        <a class="navbar-brand" href="#">NobelOptic</a>
        <div>
            <a class="btn btn-success btn-sm me-2" href="/product_create.php">تعریف کالا جدید</a>
            <a class="btn btn-success btn-sm me-2" href="/product_bulk_create.php">تعریف گروهی کالا</a>
            <a class="btn btn-outline-secondary btn-sm" href="/logout.php">خروج</a>
        </div>
    </div>
</nav>

<div class="container py-4">
    <?php if (!empty($_SESSION['flash_message'])): ?>
        <div class="alert alert-success"><?= htmlspecialchars($_SESSION['flash_message']) ?></div>
        <?php unset($_SESSION['flash_message']); ?>
    <?php endif; ?>

    <h4>جستجو و فیلتر کالاها</h4>

    <form method="get" class="row g-2 align-items-end mb-3" id="filterForm">
        <!-- hidden per_page kept in form; visible control is in toolbar above table -->
        <input type="hidden" name="per_page" id="per_page_hidden" value="<?= htmlspecialchars($per_page_raw) ?>">

        <div class="col-md-4">
            <label class="form-label">تأمین‌کننده</label>
            <select name="supplier[]" class="form-select" style="background-position:left.75rem center;" id="supplierSelect" multiple>
                <option value="all" <?= $useAllSupplier ? 'selected' : '' ?>>همه</option>
                <?php foreach ($supList as $s): ?>
                    <option value="<?= $s['id'] ?>" <?= in_array($s['id'], $suppliers) ? 'selected' : '' ?>><?= htmlspecialchars($s['name']) ?></option>
                <?php endforeach; ?>
            </select>
        </div>

        <div class="col-md-4">
            <label class="form-label">برند</label>
            <select name="brand[]" class="form-select" style="background-position:left.75rem center;" id="brandSelect" multiple>
                <option value="all" <?= $useAllBrand ? 'selected' : '' ?>>همه</option>
                <?php foreach ($brandList as $b): ?>
                    <option value="<?= $b['id'] ?>" <?= in_array($b['id'], $brands) ? 'selected' : '' ?>><?= htmlspecialchars($b['name']) ?></option>
                <?php endforeach; ?>
            </select>
        </div>

        <div class="col-md-4">
            <label class="form-label">جنس</label>
            <select name="material[]" class="form-select" style="background-position:left.75rem center;" id="materialSelect" multiple>
                <option value="all" <?= $useAllMaterial ? 'selected' : '' ?>>همه</option>
                <?php foreach ($materialList as $m): ?>
                    <option value="<?= $m['id'] ?>" <?= in_array($m['id'], $materials) ? 'selected' : '' ?>><?= htmlspecialchars($m['name']) ?></option>
                <?php endforeach; ?>
            </select>
        </div>

        <div class="col-md-12">
            <label class="form-label">جستجوی تگ (مثال: فتو 2-2)</label>
            <input name="tags" value="<?= htmlspecialchars($tagSearch) ?>" class="form-control" placeholder="کلمات را با فاصله جدا کنید">
        </div>

        <!-- hidden original buttons kept for progressive enhancement -->
        <div class="col-md-12 d-none">
            <button class="btn btn-primary mt-2" id="applyBtnHidden">اعمال فیلتر</button>
            <a class="btn btn-secondary mt-2" href="/products.php" id="clearBtnHidden">پاک کردن</a>
        </div>
    </form>

    <!-- toolbar above table: left = per-page, right = apply/clear -->
    <div class="toolbar" role="toolbar" aria-label="table toolbar">
        <div class="left">
            <select id="perPageSelect" class="form-select per-page-select" aria-label="تعداد آیتم در صفحه" style="background-position:left.75rem center;">
                <?php foreach ($perPageOptions as $opt): ?>
                    <option value="<?= $opt ?>" <?= ((string)$per_page_raw === (string)$opt) ? 'selected' : '' ?>><?= ($opt === 'all') ? 'همه' : $opt ?></option>
                <?php endforeach; ?>
            </select>
        </div>

        <div class="right">
            <button class="btn btn-primary" id="applyFiltersBtn">اعمال فیلتر</button>
            <button class="btn btn-secondary" id="clearFiltersBtn">پاک کردن</button>
        </div>
    </div>

    <div class="table-responsive">
        <table class="table table-striped table-bordered">
            <thead>
            <tr>
                <th><?= sortLink('id', '#', $sort, $order) ?></th>
                <th><?= sortLink('name', 'نام', $sort, $order) ?></th>
                <th><?= sortLink('supplier', 'تأمین‌کننده', $sort, $order) ?></th>
                <th><?= sortLink('brand', 'برند', $sort, $order) ?></th>
                <th><?= sortLink('group', 'گروه', $sort, $order) ?></th>
                <th><?= sortLink('price_base', 'قیمت اصلی', $sort, $order) ?></th>
                <th><?= sortLink('price_after', 'قیمت تخفیفی', $sort, $order) ?></th>
				<th>بروز رسانی</th>
				<th>عملیات</th>
            </tr>
            </thead>
            <tbody>
            <?php if (empty($results)): ?>
                <tr><td colspan="8" class="text-center">هیچ رکوردی یافت نشد</td></tr>
            <?php else: foreach ($results as $row): ?>
                <?php
                $supN = $supplierMap[$row['supplier_id']] ?? '-';
                $brN = $brandMap[$row['brand_id']] ?? '-';
                $groupName = $groupMap[$row['group_id']] ?? '-';
                ?>
                <tr>
                    <td><?= htmlspecialchars($row['id']) ?></td>
                    <td><?= htmlspecialchars($row['name'] ?? '-') ?></td>
                    <td><?= htmlspecialchars($supN) ?></td>
                    <td><?= htmlspecialchars($brN) ?></td>
                    <td><?= htmlspecialchars($groupName) ?></td>
                    <td><?= isset($row['price_base']) ? number_format((float)$row['price_base']) : '-' ?></td>
                    <!-- <td><?= number_format((float)$row['price_after_discount']) ?></td> -->
					<!-- <td><?= $row['supplier_id'] == 10 ? number_format((float)($row['price_after_discount'] * 0.9)) : number_format((float)$row['price_after_discount']) ?></td> -->
					<td><?= $row['supplier_id'] == 10 ? number_format((float)$row['price_after_second_discount']) : number_format((float)$row['price_after_discount']) ?></td>
                    <td><?= date_to_jalali($row['date_effective']) ?></td>
					<td>
                        <a class="edit-btn" title="ویرایش" href="/product_edit.php?id=<?= $row['id'] ?>">
                            <svg xmlns="http://www.w3.org/2000/svg" width="14" height="14" fill="#6c757d" viewBox="0 0 16 16" aria-hidden="true">
                                <path d="M12.146.854a.5.5 0 0 1 .708 0l2.292 2.292a.5.5 0 0 1 0 .708l-9.193 9.193a.5.5 0 0 1-.168.11l-4 1.5a.5.5 0 0 1-.65-.65l1.5-4a.5.5 0 0 1 .11-.168l9.193-9.193zM11.207 3L13 4.793 12.207 5.586 10.414 3.793 11.207 3zM10.5 3.707L12.293 5.5 4.5 13.293 2.707 11.5 10.5 3.707z"/>
                            </svg>
                            <span style="font-size:13px;color:#333;">ویرایش</span>
                        </a>
                    </td>
                </tr>
            <?php endforeach; endif; ?>
            </tbody>
        </table>
    </div>

    <!-- Pagination controls -->
    <?php
    // If per_page == 'all' then show single page
    if ($per_page === 'all') {
        $totalPages = 1;
        $page = 1;
    } else {
        $totalPages = max(1, (int)ceil($totalItems / $per_page));
    }
    $visibleRange = 5;
    $startPage = max(1, $page - floor($visibleRange/2));
    $endPage = min($totalPages, $startPage + $visibleRange - 1);
    if ($endPage - $startPage + 1 < $visibleRange) {
        $startPage = max(1, $endPage - $visibleRange + 1);
    }
    ?>
    <div class="d-flex justify-content-between align-items-center mt-3">
        <div>
            <small>نمایش <?= ($totalItems>0) ? (($page-1)*($per_page==='all'? $totalItems : $per_page)+1) : 0 ?> تا <?= min($totalItems, $page*($per_page==='all'? $totalItems : $per_page)) ?> از <?= $totalItems ?> مورد</small>
        </div>
        <nav aria-label="Pagination">
            <ul class="pagination mb-0">
                <li class="page-item <?= $page<=1 ? 'disabled' : '' ?>">
                    <a class="page-link" href="?<?= buildQuery(['page'=>1]) ?>" aria-label="first">« اول</a>
                </li>
                <li class="page-item <?= $page<=1 ? 'disabled' : '' ?>">
                    <a class="page-link" href="?<?= buildQuery(['page'=>max(1,$page-1)]) ?>" aria-label="prev">‹ قبلی</a>
                </li>

                <?php for ($p = $startPage; $p <= $endPage; $p++): ?>
                    <li class="page-item <?= $p==$page ? 'active' : '' ?>">
                        <a class="page-link" href="?<?= buildQuery(['page'=>$p]) ?>"><?= $p ?></a>
                    </li>
                <?php endfor; ?>

                <li class="page-item <?= $page>=$totalPages ? 'disabled' : '' ?>">
                    <a class="page-link" href="?<?= buildQuery(['page'=>min($totalPages,$page+1)]) ?>" aria-label="next">بعدی ›</a>
                </li>
                <li class="page-item <?= $page>=$totalPages ? 'disabled' : '' ?>">
                    <a class="page-link" href="?<?= buildQuery(['page'=>$totalPages]) ?>" aria-label="last">آخر »</a>
                </li>
            </ul>
        </nav>
    </div>

</div>

<script>
    (function(){
        const perPageSelect = document.getElementById('perPageSelect');
        const perPageHidden = document.getElementById('per_page_hidden');
        const filterForm = document.getElementById('filterForm');
        const applyBtn = document.getElementById('applyFiltersBtn');
        const clearBtn = document.getElementById('clearFiltersBtn');

        if (!filterForm) return;

        // When per-page changes, update hidden and submit immediately
        if (perPageSelect && perPageHidden) {
            perPageSelect.addEventListener('change', function(){
                perPageHidden.value = this.value;
                filterForm.submit();
            });
        }

        if (applyBtn) {
            applyBtn.addEventListener('click', function(e){
                e.preventDefault();
                if (perPageSelect && perPageHidden) perPageHidden.value = perPageSelect.value;
                filterForm.submit();
            });
        }

        if (clearBtn) {
            clearBtn.addEventListener('click', function(e){
                e.preventDefault();
                window.location.href = '/products.php';
            });
        }

        // 'all' option behavior for multi-selects (no disabling, just mutual exclusivity)
        function setupAllOption(selectId){
            const sel = document.getElementById(selectId);
            if(!sel) return;
            sel.addEventListener('change', function(){
                const opts = Array.from(sel.options);
                const allOpt = opts.find(o => o.value === 'all');
                if (!allOpt) return;
                if (allOpt.selected) {
                    opts.forEach(o => { if (o.value !== 'all') o.selected = false; });
                } else {
                    allOpt.selected = false;
                }
            });
        }
        setupAllOption('supplierSelect');
        setupAllOption('brandSelect');
        setupAllOption('materialSelect');

        // Submit on Enter in tags input
        const tagsInput = document.querySelector('input[name="tags"]');
        if (tagsInput) {
            tagsInput.addEventListener('keydown', function(e){
                if (e.key === 'Enter') {
                    e.preventDefault();
                    if (perPageSelect && perPageHidden) perPageHidden.value = perPageSelect.value;
                    filterForm.submit();
                }
            });
        }

        // ensure hidden per_page matches visible on load
        if (perPageSelect && perPageHidden) perPageHidden.value = perPageSelect.value;

    })();
</script>
</body>
</html>
