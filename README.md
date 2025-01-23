# サービスクラスとリポジトリパターン
サービスクラスとリポジトリクラス（リポジトリパターン）について、Laravelを使った具体例を交えて説明します。  

## 1. サービスクラスとリポジトリクラスとは？
- **サービスクラス**  
  ビジネスロジックをコントローラーから切り離して、専用のクラスにまとめる設計手法です。  
  例: 注文の計算処理や複雑な条件に基づくデータ操作を一箇所にまとめる。  

- **リポジトリクラス（リポジトリパターン）**  
  データアクセスロジック（データベースとのやり取り）を専用のクラスに切り出すパターンです。  
  例: Eloquentモデルを直接操作するのではなく、リポジトリを通じてデータを取得・保存する。

---

## 2. 使い方を具体例で解説  

### 例: 商品を注文するシステム  
- **要件**  
  1. 商品を注文するAPIを作成する。  
  2. 在庫チェックを行い、注文データを保存する。  

---

### MVCでの「Fatコントローラー」例  
コントローラーにすべてのロジックを記述すると以下のようになります：

```php
class OrderController extends Controller
{
    public function placeOrder(Request $request)
    {
        // 在庫チェック
        $product = Product::find($request->product_id);
        if ($product->stock < $request->quantity) {
            return response()->json(['error' => '在庫不足'], 400);
        }

        // 在庫を減らす
        $product->stock -= $request->quantity;
        $product->save();

        // 注文データを保存
        $order = new Order();
        $order->product_id = $request->product_id;
        $order->quantity = $request->quantity;
        $order->total_price = $product->price * $request->quantity;
        $order->save();

        return response()->json(['success' => '注文完了']);
    }
}
```

このコードは以下の問題があります：  
1. **可読性が低い**：複数の責務（在庫チェック、在庫更新、注文作成）が混在している。  
2. **テストが困難**：ビジネスロジックを部分的にテストするのが難しい。  

---

### サービスクラスとリポジトリクラスを使った改善例  

#### **リポジトリクラス**
データアクセスロジックをここに集約します。

```php
namespace App\Repositories;

use App\Models\Product;

class ProductRepository
{
    public function find(int $id)
    {
        return Product::find($id);
    }

    public function reduceStock(Product $product, int $quantity)
    {
        $product->stock -= $quantity;
        $product->save();
    }
}
```

#### **サービスクラス**
ビジネスロジックをここに集約します。

```php
namespace App\Services;

use App\Repositories\ProductRepository;
use App\Models\Order;

class OrderService
{
    private $productRepository;

    public function __construct(ProductRepository $productRepository)
    {
        $this->productRepository = $productRepository;
    }

    public function placeOrder(int $productId, int $quantity)
    {
        // 在庫チェック
        $product = $this->productRepository->find($productId);
        if ($product->stock < $quantity) {
            throw new \Exception('在庫不足');
        }

        // 在庫を減らす
        $this->productRepository->reduceStock($product, $quantity);

        // 注文データを保存
        $order = new Order();
        $order->product_id = $productId;
        $order->quantity = $quantity;
        $order->total_price = $product->price * $quantity;
        $order->save();

        return $order;
    }
}
```

#### **コントローラー**
コントローラーは入力の受け渡しだけに専念します。

```php
namespace App\Http\Controllers;

use App\Services\OrderService;
use Illuminate\Http\Request;

class OrderController extends Controller
{
    private $orderService;

    public function __construct(OrderService $orderService)
    {
        $this->orderService = $orderService;
    }

    public function placeOrder(Request $request)
    {
        try {
            $order = $this->orderService->placeOrder(
                $request->product_id,
                $request->quantity
            );
            return response()->json(['success' => '注文完了', 'order' => $order]);
        } catch (\Exception $e) {
            return response()->json(['error' => $e->getMessage()], 400);
        }
    }
}
```

---

## 3. メリット・デメリット  

### メリット
1. **責務の分離**  
   - コントローラー、サービス、リポジトリそれぞれが特定の責務に集中するため、コードが整理されて可読性が向上します。

2. **テストが容易**  
   - サービスクラスやリポジトリクラスを個別にテストできるため、ユニットテストが書きやすくなります。

3. **再利用性の向上**  
   - サービスやリポジトリを他のコントローラーやタスクスケジューラなどでも再利用できます。

### デメリット
1. **コードの分散**  
   - クラスが増えるため、初見のエンジニアには追いづらい場合があります。

2. **過剰設計のリスク**  
   - 小規模プロジェクトや単純な処理では、設計が複雑になりすぎる可能性があります。

---

## まとめ
サービスクラスとリポジトリクラスを使うことで、コードの責務を明確に分け、可読性やテスト容易性を向上させることができます。ただし、プロジェクトの規模や複雑さに応じて適切に導入することが重要です。  

必要に応じてサンプルコードを調整して、自分のプロジェクトに取り入れてみてください！
