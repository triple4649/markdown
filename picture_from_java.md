## Java で画像を読み込むサンプルコード

Java では **`javax.imageio.ImageIO`** を使うと、PNG・JPG・GIF・BMP など多くの画像フォーマットを簡単に読み込めます。  
以下では 3 つの読み込みパターン（ファイル、リソース、URL）を示し、さらに Swing で表示する例も付けています。

> **ポイント**  
> * `ImageIO.read(...)` は **`BufferedImage`** を返します。  
> * 例外は `IOException` で捕捉します。  
> * 画像が見つからない場合やサポート外フォーマットの場合は `null` を返します（`IOException` ではなく `null` になるケースもあります）。

---

### 1. ファイルパスから読み込む

```java
import javax.imageio.ImageIO;
import java.awt.image.BufferedImage;
import java.io.File;
import java.io.IOException;

public class ImageLoaderFile {
    public static void main(String[] args) {
        // 画像ファイルへのパス
        String imagePath = "C:/images/sample.png";  // Windows 例
        // File オブジェクトを作成
        File imageFile = new File(imagePath);

        try {
            // ImageIO で読み込み
            BufferedImage img = ImageIO.read(imageFile);

            if (img == null) {
                System.err.println("画像を読み込めませんでした (null 取得)。");
                return;
            }

            System.out.println("画像サイズ: " + img.getWidth() + " x " + img.getHeight());
            // ここで img を使って処理を続けます
        } catch (IOException e) {
            System.err.println("画像読み込み中にエラーが発生しました:");
            e.printStackTrace();
        }
    }
}
```

> **備考**  
> * `File` を直接渡す場合、ファイルが存在しないと `IOException` がスローされます。  
> * `ImageIO.read` は内部で `ImageReader` を選択してデコードするので、フォーマットに応じて自動で最適なデコーダが使われます。

---

### 2. JAR 内のリソース（パッケージリソース）から読み込む

```java
import javax.imageio.ImageIO;
import java.awt.image.BufferedImage;
import java.io.IOException;
import java.io.InputStream;

public class ImageLoaderResource {
    public static void main(String[] args) {
        // JAR 内のパス（先頭に / を付けるとクラスパスのルートから検索）
        String resourcePath = "/images/icon.png";

        try (InputStream is = ImageLoaderResource.class.getResourceAsStream(resourcePath)) {
            if (is == null) {
                System.err.println("リソースが見つかりません: " + resourcePath);
                return;
            }

            BufferedImage img = ImageIO.read(is);
            if (img == null) {
                System.err.println("画像を読み込めませんでした (null 取得)。");
                return;
            }

            System.out.println("リソース画像サイズ: " + img.getWidth() + " x " + img.getHeight());
        } catch (IOException e) {
            System.err.println("画像読み込み中にエラーが発生しました:");
            e.printStackTrace();
        }
    }
}
```

> **ポイント**  
> * `Class#getResourceAsStream` は JAR 内のリソースを `InputStream` で取得します。  
> * `try-with-resources` で自動クローズしています。  
> * JAR からの読み込みはファイルシステムからの読み込みよりも遅くなる場合がありますが、パッケージに埋め込む際は便利です。

---

### 3. URL（Web 上の画像）から読み込む

```java
import javax.imageio.ImageIO;
import java.awt.image.BufferedImage;
import java.io.IOException;
import java.net.URL;

public class ImageLoaderUrl {
    public static void main(String[] args) {
        // Web 上の画像 URL
        String imageUrl = "https://example.com/images/sample.jpg";

        try {
            URL url = new URL(imageUrl);
            BufferedImage img = ImageIO.read(url);

            if (img == null) {
                System.err.println("画像を読み込めませんでした (null 取得)。");
                return;
            }

            System.out.println("Web 画像サイズ: " + img.getWidth() + " x " + img.getHeight());
        } catch (IOException e) {
            System.err.println("画像読み込み中にエラーが発生しました:");
            e.printStackTrace();
        }
    }
}
```

> **備考**  
> * `ImageIO.read(URL)` は内部で `URLConnection` を開き、ストリームを取得します。  
> * ネットワーク遅延や接続失敗時に `IOException` がスローされます。

---

### 4. Swing で表示してみる（実際に画面に描画）

```java
import javax.swing.*;
import javax.imageio.ImageIO;
import java.awt.*;
import java.awt.image.BufferedImage;
import java.io.IOException;
import java.io.File;

public class ImageDisplayDemo {
    public static void main(String[] args) {
        SwingUtilities.invokeLater(() -> {
            JFrame frame = new JFrame("Image Display Demo");
            frame.setDefaultCloseOperation(JFrame.EXIT_ON_CLOSE);

            try {
                // ファイルから読み込み（必要に応じてリソースや URL に変更可）
                BufferedImage img = ImageIO.read(new File("C:/images/sample.png"));

                if (img == null) {
                    throw new IOException("画像を読み込めませんでした (null 取得)。");
                }

                // ImageIcon にラップして JLabel へ設定
                JLabel label = new JLabel(new ImageIcon(img));
                frame.add(label, BorderLayout.CENTER);
            } catch (IOException e) {
                e.printStackTrace();
                JLabel errorLabel = new JLabel("画像読み込みエラー");
                frame.add(errorLabel, BorderLayout.CENTER);
            }

            frame.pack();           // コンテンツに合わせてサイズ調整
            frame.setLocationRelativeTo(null); // 画面中央に配置
            frame.setVisible(true);
        });
    }
}
```

> **Tips**  
> * `ImageIcon` は `BufferedImage` を受け取るコンストラクタがあります。  
> * 画像サイズが大きい場合は `frame.pack()` で自動調整するか、`JScrollPane` に入れてスクロール可能にするのがおすすめです。

---

## まとめ

| 目的 | コードの抜粋 | ポイント |
|------|--------------|----------|
| ファイルから読み込み | `ImageIO.read(new File(path))` | `File` で直接パス指定 |
| リソース（Jar 内） | `ImageIO.read(getClass().getResourceAsStream("/images/img.png"))` | クラスパスから取得 |
| URL から読み込み | `ImageIO.read(new URL("https://…"))` | ネットワーク経由 |
| Swing で表示 | `new ImageIcon(img)` で `JLabel` に設定 | GUI での描画 |

`ImageIO.read` は非常に汎用的で使い勝手が良いので、まずはこのメソッドから始めると良いでしょう。さらに画像を加工したい場合は `BufferedImage` でピクセル単位で操作するか、`Graphics2D` を使った描画が可能です。ぜひ試してみてください！
