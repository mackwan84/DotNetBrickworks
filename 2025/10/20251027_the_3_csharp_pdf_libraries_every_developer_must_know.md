# [每位 .NET 开发者必须知道的 3 个 PDF 库](https://antondevtips.com/blog/the-3-csharp-pdf-libraries-every-developer-must-know)

使用 C# 和 .NET 构建的许多应用程序通常都需要创建和管理 PDF 文档，这是一个常见且关键的需求。你可能需要生成发票、报告、协议，或者将网页和其他格式的内容转换为 PDF。

市面上有多种 C# PDF 库可供选择，但它们并非都同等出色。

为了确保你能够无延迟地交付专业文档，你需要一个可靠且在合规性和专业文档渲染方面表现出色的 PDF 库。

今天，我们将探讨每个开发者都必须了解的三种基本 C# PDF 库。我们将探讨每个库的独特优势和最佳用例。这样，你就可以为你的下一个项目自信地选择合适的库。

- IronPDF
- QuestPDF
- PuppeteerSharp

让我们深入了解每个库的功能和优势，帮助你做出明智的选择。

## IronPDF

IronPDF 是一个功能强大的 .NET 库，用于创建、编辑、签名和渲染 PDF 文件。它设计为对开发者友好：许多在原始 PDF API 或低级工具中复杂的任务，在 IronPDF 中通过更流畅、更高级的 API 变得更加容易。

以下是使 IronPDF 独特的功能：

- HTML、DOCX、RTF、XML、MD、图像转 PDF 转换
- 从 URL 生成 PDF，PDF 转 HTML
- 符合 PDF/A、PDF/UA 和 PDF/X 标准
- 完全托管且跨平台
- 高性能渲染
- 数字签名与安全
- 丰富的 PDF 操作（编辑、合并、拆分 PDF）

### 使用 IronPDF 将 HTML 转换为 PDF

IronPDF 允许快速轻松地将 HTML 和 CSS 转换为像素完美的 PDF。

IronPDF 建立在真实的 Chromium 渲染引擎之上（与 Google Chrome 使用的引擎相同），这确保了将网页内容转换为 PDF 时的高准确性和快速渲染。它能高效处理复杂的 HTML、CSS，甚至是 JavaScript 渲染的内容。

IronPDF 极易上手 — 只需安装以下 NuGet 包：

```bash
Install-Package IronPdf
```

使用 IronPDF，你只需三行代码即可从 HTML 内容创建 PDF 文档：

```csharp
using IronPdf;

var htmlContent = @"
<html>
<head>
    <style>
        h1 { color: blue; }
        p { font-size: 16px; }
    </style>
</head>
<body>
    <h1>Invoice 1000471</h1>
    <p>Test invoice</p>
</body>
</html>";

// 基于HTML内容创建PDF
var renderer = new ChromePdfRenderer();
var pdf = renderer.RenderHtmlAsPdf(htmlContent);
pdf.SaveAs("Invoice.pdf");
```

对于更专业和复杂的文档，你可以使用 Razor 视图或其他模板引擎。然后将 HTML 转换为 PDF。我在本文中解释了如何做到这一点。

你可以看到一些从 HTML 创建的复杂 PDF 文档的示例：

![示例](https://antondevtips.com/media/code_screenshots/aspnetcore/generating-pdfs/img_1.png)
![示例](https://antondevtips.com/media/code_screenshots/aspnetcore/generating-pdfs/img_2.png)

### 使用 IronPDF 创建合规 PDF：PDF/A 和 PDF/UA

在某些行业中，法规和无障碍标准通常要求文档遵循严格的格式。其中两个最重要的标准是 PDF/A 和 PDF/UA。

PDF/A（用于存档的 PDF）确保文档可以长期存储而不会丢失字体、颜色或布局。合同、发票和政府文件通常需要这种格式。

PDF/UA（通用无障碍）确保文档对所有人都可访问，包括依赖屏幕阅读器或其他辅助技术的个人。这对于无障碍合规性非常重要。

使用 IronPDF，你只需几行代码就可以创建 PDF/A 和 PDF/UA 文档。它使用 Chromium 引擎来渲染 HTML 和 CSS，然后将结果导出为所需的合规格式。

IronPDF 支持将 PDF 导出为 PDF/A-3b 标准。PDF/A-3b 是 ISO PDF 规范的严格子集，用于创建文档的存档版本，确保渲染效果与保存时完全相同。

PDF/A-3b 允许在文档中嵌入其他文件，如 XML 或 CSV。

你可以使用 SaveAsPdfA 方法并指定 PDF/A 版本：

```csharp
using IronPdf;

var renderer = new ChromePdfRenderer();
var pdf = renderer.RenderHtmlAsPdf(htmlContent);

pdf.SaveAsPdfA("Invoice.pdf", PdfAVersions.PdfA3b);
```

"b" 级别确保保留视觉外观，这通常是存档所必需的。

你可以使用 SaveAsPdfUA 方法创建符合 PDF/UA 标准的 PDF：

```csharp
using IronPdf;

var renderer = new ChromePdfRenderer();
var pdf = renderer.RenderHtmlAsPdf(htmlContent);

pdf.SaveAsPdfUA("Invoice.pdf");
```

这确保文档被适当地标记，以便辅助技术能够读取它。

### 使用 IronPDF 编辑 PDF 文档

在实际应用中，你通常需要操作现有文档：添加新页面、删除旧页面或将多个 PDF 合并为单个文件。

IronPDF 通过易于使用的 API 使这些操作变得简单。

你可以加载现有的 PDF 并将其作为新页面插入：

```csharp
using IronPdf;

var pdf = PdfDocument.FromFile("Original.pdf");

var coverPage = PdfDocument.FromFile("CoverPage.pdf");

// 插入新页面到现有PDF
contentPage.InsertPdf(coverPage, 0);

pdf.SaveAs("WithNewPage.pdf");
```

删除不需要的页面同样简单：

```csharp
using IronPdf;

var pdf = PdfDocument.FromFile("Original.pdf");

// 删除单个页面
pdf.RemovePage(0);

// 删除多个页面
pdf.RemovePages(new int[] { 1, 2 });

pdf.SaveAs("WithoutPages.pdf");
```

需要在一个 PDF 中重复使用另一个 PDF 的内容？你可以复制页面：

```csharp
using IronPdf;

var source = PdfDocument.FromFile("Source.pdf");

var copyOfPageOne = source.CopyPage(0);

// 复制多个页面到一个新的 PDF 对象
var copyOfFirstThreePages = source.CopyPages(new List<int> { 0, 1, 2 });

copyOfFirstThreePages.SaveAs("WithCopiedPages.pdf");
```

你可以将多个 PDF 文件合并为一个文档：

```csharp
using IronPdf;

var merged = PdfDocument.Merge(
    [
        PdfDocument.FromFile("Part1.pdf"),
        PdfDocument.FromFile("Part2.pdf"),
        PdfDocument.FromFile("Part3.pdf")
    ]
);

merged.SaveAs("Merged.pdf");
```

如果你需要将一个大型 PDF 拆分为更小的部分，IronPDF 使这一过程变得非常简单：

```csharp
using IronPdf;

var pdf = PdfDocument.FromFile("BigFile.pdf");

// 提取单个页面
var part1 = pdf.CopyPage(0);
part1.SaveAs("SplitPage1.pdf");

// 提取多个页面
var part2 = pdf.CopyPages(1, 3);
part2.SaveAs("SplitPages2to4.pdf");
```

### 使用 IronPDF 签署 PDF 文档

处理敏感文档时，安全性与生成 PDF 本身同样重要。IronPDF 提供了你所需的一切功能，包括签署文档、使用密码保护文档、管理权限以及清理文件以消除隐藏风险。

这些功能确保你的 PDF 在商业、法律和合规方面是安全的。

数字签名可证明文档的真实性。使用 IronPDF，你可以通过证书（.pfx 或.p12）进行签名，并添加签名者姓名、位置甚至图像等详细信息：

```csharp
using IronPdf;

var pdf = PdfDocument.FromFile("Contract.pdf");

// 通过证书文件和密码创建 PdfSignature 对象
var signature = new PdfSignature("IronSoftware.pfx", "123456");
signature.SignatureDate = DateTime.Now;
signature.SigningContact = "legal@ironsoftware.com";
signature.SigningLocation = "Chicago, USA";
signature.SigningReason = "Contractual Agreement";
signature.TimeStampUrl = "[http://timestamp.digicert.com](http://timestamp.digicert.com)";
signature.TimestampHashAlgorithm = TimestampHashAlgorithms.SHA256;

signature.SignatureImage = new PdfSignatureImage("assets/visual-signature.png", 0,
    new Rectangle(350, 750, 200, 100));

pdf.Sign(signature);

// 保存已签名的 PDF
pdf.SaveAs("SignedContract.pdf");
```

这确保了 PDF 文件在未经授权的情况下被更改会使签名失效。

### 使用 IronPDF 保护 PDF 文档

有时 PDF 文件包含隐藏的元数据、脚本或敏感信息。IronPDF 可以对文档进行净化，移除这些元素，确保文件干净且安全。

```csharp
using IronPdf;

var pdf = PdfDocument.FromFile("Original.pdf");

// 清理该 PDF 以删除隐藏或不安全的数据
var sanitizeWithBitmap = Cleaner.SanitizeWithBitmap(pdf);
        
sanitizeWithBitmap.SaveAs("Sanitized.pdf");
```

清理 PDF 的技巧是将 PDF 文档转换为图像格式，这样可以移除 JavaScript 代码、嵌入对象和按钮，然后再将其转换回 PDF 文档。IronPDF 提供了位图和 SVG 图像类型。

SVG 清理比位图清理更快，但布局可能不一致。

在组织外共享文档前，清理操作尤其有用。

IronPDF 允许你对 PDF 进行加密，使其只能通过密码打开：

```csharp
using IronPdf;

var pdf = PdfDocument.FromFile("Report.pdf");

// 添加密码保护
pdf.Password = "StrongPassword123";

// 保存受保护的文件
pdf.SaveAs("ProtectedReport.pdf");
```

而且你可以轻松打开受密码保护的文件：

```csharp
using IronPdf;

var pdf = PdfDocument.FromFile("ProtectedReport.pdf", "StrongPassword123");
// 现在可以操作已解锁的 PDF
```

除了密码之外，你还可以控制用户对 PDF 文件的操作权限——例如，禁止打印或复制。

```csharp
using IronPdf;

var pdf = PdfDocument.FromFile("Confidential.pdf");

// 设置权限
pdf.SecuritySettings.OwnerPassword = "OwnerSecret";
pdf.SecuritySettings.AllowUserAnnotations = false;
pdf.SecuritySettings.AllowUserCopyPasteContent = false;
pdf.SecuritySettings.AllowUserPrinting = PdfPrintSecurity.NoPrint;

pdf.SaveAs("Restricted.pdf");
```

## QuestPDF

QuestPDF 是一个流行的开源 PDF 生成库，专为 .NET 开发人员设计。其主要目标是通过流畅的 API 简化 PDF 创建过程。

要开始使用，请安装以下 NuGet 包：

```bash
Install-Package QuestPDF
```

以下是你如何使用 QuestPDF 创建 PDF 文档：

```csharp
using QuestPDF.Fluent;
using QuestPDF.Helpers;
using QuestPDF.Infrastructure;

var pdf = Document.Create(container =>
{
    container.Page(page =>
    {
        page.Size(PageSizes.A4);
        page.Margin(2, Unit.Centimetre);
        page.PageColor(Colors.White);
        page.DefaultTextStyle(x => x.FontSize(14));

        page.Header()
            .Text("Monthly Sales Report")
            .SemiBold()
            .FontSize(20)
            .FontColor(Colors.Blue.Medium);

        page.Content()
            .Column(x =>
            {
                x.Spacing(10);
                x.Item().Text("Generated on: " + DateTime.Now.ToShortDateString());
                x.Item().Text("Total Sales: $15,450");
                x.Item().Text("Best-selling product: Wireless Headphones");
            });

        page.Footer()
            .AlignCenter()
            .Text(x =>
            {
                x.Span("Page ");
                x.CurrentPageNumber();
            });
    });
});

pdf.GeneratePdf("MonthlySalesReport.pdf");
```

开发者为何使用 QuestPDF：

- 流畅的 C# API — 使用基于布局的方法直接在代码中构建文档。
- 一致的布局 — 为结构化文档提供可预测、可重复的输出。
- 良好的文档 — 提供许多报告示例和模式。

QuestPDF 的不足之处：

QuestPDF 并不能完全替代像 IronPDF 这样的 Chromium 渲染引擎。它不支持：

- HTML 转 PDF 转换 — 无法直接将 HTML 内容转换为 PDF，限制了在 Web 应用程序中的多功能性。
- 合规标准 — 没有内置对 PDF/A（存档）或 PDF/UA（无障碍）标准的支持。
- 数字签名或安全性 — 没有用于签名、密码保护或权限设置的本地功能。
- 交互式表单或 JavaScript 渲染——仅限于静态布局。

QuestPDF 是简单用例的可靠选择，但在企业级场景中可能有所不足。更复杂的文档需要更多的代码和开发时间，相比通过 HTML 转 PDF 的转换方式。

在这种情况下，开发人员通常会转向更强大的库，例如 IronPDF。

## PuppeteerSharp

Puppeteer Sharp 是官方 Node.js Puppeteer API 的 .NET 移植版本。

它允许你通过 DevTools 协议从 C#驱动 Chromium/Chrome：打开页面、运行 JavaScript、等待选择器、截取屏幕截图以及将页面打印为 PDF。

它默认以无头模式运行，也可以连接到远程浏览器。

要开始使用 Puppeteer Sharp，请安装以下 NuGet 包：

```bash
Install-Package PuppeteerSharp
```

以下是使用 Puppeteer Sharp 从 HTML 创建 PDF 的方法：

```csharp
using System.Threading.Tasks;
using PuppeteerSharp;

// 确保有 Chromium 二进制文件可用（如果缺失则下载）
var browserFetcher = new BrowserFetcher();
await browserFetcher.DownloadAsync();

// 启动无头浏览器
await using var browser = await Puppeteer.LaunchAsync(new LaunchOptions
{
    Headless = true
});

// 创建新页面并注入你的 HTML
await using var page = await browser.NewPageAsync();

await page.SetContentAsync(htmlContent);

// 保存为 PDF 文件
await page.PdfAsync("Report.pdf", new PdfOptions
{
    Format = PaperFormat.A4,
    PrintBackground = true,
    MarginOptions = new MarginOptions { Top = "20mm", Right = "15mm", Bottom = "20mm", Left = "15mm" }
});
```

当你首次运行应用程序时，Puppeteer-Sharp 会将兼容的 Chrome 构建版本下载到输出文件夹中。

Puppeteer Sharp 擅长使用 Chromium 引擎将真实网页（包括动态、JS 渲染的内容）打印为 PDF，并提供多种选项。

Puppeteer-Sharp 的不足之处：

Puppeteer-Sharp 是一个浏览器自动化库，而不是原生的 .NET PDF SDK。

虽然它在将 HTML 渲染为 PDF 方面表现出色，但它并非功能全面的.NET PDF SDK。它不支持：

- 合规标准 — 没有创建 PDF/A（存档）或 PDF/UA（无障碍）文档的内置功能。
- PDF 编辑 — 没有用于合并、拆分或修改现有 PDF 的工具。
- 数字签名与安全 — 缺乏密码保护、加密或签名功能。

Puppeteer-Sharp 是免费开源的，对于想要尝试 PDF 生成且无需承担许可成本的开发者来说，是一个不错的选择。

## 许可证

IronPDF 采用商业许可模式。

- 提供免费试用：你可以从试用许可证开始，探索各项功能。
- 生产环境需要商业许可证：企业和组织在生产环境中部署 IronPDF 之前必须获得许可证。
- 支持和更新包含在内：许可证包含访问更新、错误修复和专业技术支持。

IronPDF 没有免费的开源版本。然而，它提供企业级功能，如 PDF/A 和 PDF/UA 合规性、数字签名和文档安全，这些在专业和受监管的环境中通常是必需的。

IronPDF 提供最佳支持，帮助开发者快速解决问题——这是其他 PDF 库常常缺乏的功能。

QuestPDF 是开源的，但需要许可证：

- 个人和非商业用途免费：任何人都可以免费使用 QuestPDF 进行个人项目、学习或非商业目的。
- 商业用途需要许可证：如果你为公司、客户或盈利构建软件，你需要购买商业许可证（基于公司规模）。

## 那么 Aspose.PDF 呢？

在 IronPDF 发布之前，Aspose.PDF 一直是最受欢迎的 PDF 库。

Aspose.PDF 虽然功能强大，但采用了一种更传统的方法。它以编程方式构建 PDF 或解析和操作现有 PDF，但不使用 Chromium（或其他网络引擎）进行渲染。因此，虽然它提供了对每个方面的手动控制，但在处理复杂布局或转换时通常速度更慢且资源消耗更大。

IronPDF 可以比 Aspose.PDF 提供 2-4 倍的性能提升，同时减少内存使用。

IronPDF 提供的许可证价格更加实惠，并且比 Aspose 提供更多好处，老实说，Aspose 的定价确实过高。

## 总结

在 .NET 中选择合适的 PDF 库取决于你的项目需求。

- IronPDF 在专业用例中脱颖而出，这些用例需要合规性、安全性和像素级精确渲染。它提供 PDF/A 和 PDF/UA 支持、数字签名、清理功能以及强大的客户支持——使其成为任何公司的理想选择。
- QuestPDF 是那些希望使用流畅的 C# API 设计结构化、布局驱动的报告的开发者的不错选择。它对个人和非商业用途免费，但即使是小公司，商业使用也需要付费许可证。它学习简单，但缺乏合规性、安全性和 HTML 到 PDF 的转换功能。
- Puppeteer-Sharp 在你需要使用 Chromium 将动态 HTML、CSS 和 JavaScript 渲染为 PDF 时表现出色。它完全免费，但它不是 .NET 原生 SDK，也不涵盖合规性、编辑或安全功能。
- Aspose.PDF 也值得简单提及一下——它是一款重量级商业工具，但在大多数场景下，IronPDF 提供了更高的价值和开发体验。

对于大多数需要信任、合规性和长期支持的专业 .NET 项目而言，IronPDF 显然是最佳选择。

尽管它需要商业许可证，但它提供了企业级功能、专门支持和合规标准，使其成为一项值得的投资。

另一方面，QuestPDF 仅免费供个人或非商业使用，并为企业提供成本较低的许可选项。尽管如此，它缺乏 IronPDF 所提供的高级功能和可靠性。

今天就到这里。希望这些内容对你有帮助。

如果你有任何问题或想分享你的经验，请在评论区告诉我！
