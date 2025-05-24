在开源领域，可将 PDF 精准转成 Markdown 或 LaTeX 的方案，近两年出现了几条“主流路线”：

1. 直接端到端的转换库（Marker／MinerU／pdftolatex 等）；
2. “PDF → 结构化 XML/JSON → 二次渲染” 的流水线（GROBID＋Pandoc 等）；
3. 基于视觉 Transformer 的端到端 OCR（Meta Nougat）。
   综合活跃度、准确度与许可证来看，**Markdown 首选 MinerU；LaTeX 首选 Nougat**。下文分两部分做详细对比，并给出落地建议与快速上手命令。

## 一、PDF → Markdown：主流开源项目对比

| 项目                              | 语言           | 许可证        | 关键特点                                       | Star 数                                |
| ------------------------------- | ------------ | ---------- | ------------------------------------------ | ------------------------------------- |
| **MinerU**                      | Python       | AGPL-3.0   | 多模型融合，自动抽取表格/图像，支持 JSON/MD 双输出             | 34k([GitHub][1])                      |
| **Marker**                      | Python       | MIT        | “结构化抽取 + LLM 增强”流程，可自定义 JSON Schema        | 2k+([GitHub][2])                      |
| **pdf2md (opengovsg)**          | Node         | MIT        | 纯 JS，CLI/库双用，轻量依赖 PDF.js                   | 1k+([GitHub][3])                      |
| **pdf-to-markdown (jzillmann)** | Node         | MIT        | 早期项目，浏览器 Demo，可嵌入 Web                      | ⭐600+([GitHub][4])                    |
| **Vision-Parse**                | Python       | MIT        | 利用 Vision-LLM，每页渲染成图后 OCR→Markdown，适合含公式文档 | Reddit 社区实验([Reddit][5])              |
| **GROBID→Pandoc**               | Java+Haskell | Apache-2.0 | PDF→TEI XML→Pandoc 转 Markdown，适合科研论文批量管道   | GROBID 8k ★([GitHub][6], [Pandoc][7]) |

**结论—Markdown 场景**

* **准确率/格式还原**：MinerU 在自动识别表格、页眉/页脚消除方面效果最好；Marker 易扩展，可在公司私有 LLM 上“加一跳”提质。
* **部署与依赖**：pdf2md 与 pdf-to-markdown 体量最小，Node 生态即可；MinerU 带 GPU OCR，但也提供 Docker 镜像一键跑。
* **大规模流水线**：科研批处理可用 GROBID→Pandoc；它生成结构化 TEI，再由 Pandoc 渲染 Markdown，便于后续数据挖掘。

## 二、PDF → LaTeX：主流开源项目对比

| 项目                                  | 核心技术                                          | 许可证        | 适用文档              | Star 数                       |
| ----------------------------------- | --------------------------------------------- | ---------- | ----------------- | ---------------------------- |
| **Nougat (Meta)**                   | Vision Transformer OCR，输出 MultiMarkdown/LaTeX | MIT        | 学术论文、公式密集稿件       | 9k+([GitHub][8])             |
| **pdftolatex (vinaykanigicherla)**  | OpenCV + Tesseract                            | MIT        | 文字为主的讲义/模板        | ⭐88([GitHub][9])             |
| **pdf2latex-converter (mcpeixoto)** | Python 图像分割 + rule-based                      | MIT        | 简单排版 PDF          | ⭐50([GitHub][10])            |
| **Paper2LaTeX**                     | OpenCV 解析手绘图                                  | MIT        | 图形/曲线转 LaTeX TikZ | ⭐200([GitHub][11])           |
| **GROBID + Pandoc**                 | PDF→TEI→LaTeX                                 | Apache-2.0 | 结构化科研论文           | 同上([GitHub][6], [Pandoc][7]) |
| **PDF2LaTeX-102**                   | 公开评测数据集                                       | CC-BY-4.0  | 102 页 GT，对比算法效果   | 数据集([GitHub][12])            |

**结论—LaTeX 场景**

* **精度**：Nougat 针对公式、表格训练，在学术 PDF 上可恢复 85 %+ LaTeX 结构；可再用社区微调版 `nougat-latex-ocr` 针对公式细节做增量训练([GitHub][13])。
* **轻量脚本**：pdftolatex、pdf2latex-converter 基于传统 CV，准确率有限，但依赖少，适合在无 GPU 环境快速跑。
* **流水线替代**：若需批量抽元数据、排版一致，可走 GROBID→Pandoc→LaTeX 的多步流程。

## 三、落地建议

1. **企业/私有部署**

   * **Markdown**：推荐 MinerU（Docker/AGPL）。如需 MIT 许可证，可用 Marker，并在 “LLM 增强” 阶段接本地 Qwen/Llama 改写段落。
   * **LaTeX**：部署 Nougat GPU 推理服务；若 GPU 受限，可先用 Nougat-Small 模型或调用 Hugging Face 推理 API 本地缓存。

2. **质量提升小技巧**

   * **后处理**：Nougat 默认输出 `.mmd`，可 `pandoc file.mmd -o file.tex` 转 LaTeX；Markdown 同理。Pandoc 对公式、表格兼容性最好([Pandoc][7])。
   * **表格/图片**：MinerU & Marker 会提取图片到独立文件夹；若需纯文本，可在 CLI 中加 `--no-images` 开关。
   * **评测**：用 PDF2LaTeX-102 数据集可快速打分 BLEU/Exact Match，量化不同模型收益([GitHub][12])。

3. **快速体验命令**

```bash
# MinerU -> Markdown
docker run -v $PWD:/data opendatalab/mineru \
       --input /data/example.pdf --output /data/out.md

# Nougat -> LaTeX
git clone https://github.com/facebookresearch/nougat && cd nougat
pip install -r requirements.txt
python nougat.py --pdf example.pdf --out example.mmd
pandoc example.mmd -o example.tex
```

## 四、后续可关注的方向

* **多语言支持**：Nougat 目前主攻英语与拉丁字符；中文/阿拉伯文场景需自行微调。
* **增量编辑**：Marker 支持“结构化 JSON + 模板渲染”；可将抽取结果写入企业知识库，方便二次消费。
* **半自动校正**：结合 LLM（如 GPT-4o）对低置信度段落做增量修订，可显著降低人工校对成本。

> 采用以上方案即可在 Markdown／LaTeX 双链路上获得业界最优的开源转换效果，并保持对私有数据与许可证的充分掌控。

[1]: https://github.com/opendatalab/MinerU "GitHub - opendatalab/MinerU: A high-quality tool for convert PDF to Markdown and JSON.一站式开源高质量数据提取工具，将PDF转换成Markdown和JSON格式。"
[2]: https://github.com/VikParuchuri/marker "GitHub - VikParuchuri/marker: Convert PDF to markdown + JSON quickly with high accuracy"
[3]: https://github.com/opengovsg/pdf2md "GitHub - opengovsg/pdf2md: A PDF to Markdown converter"
[4]: https://github.com/jzillmann/pdf-to-markdown?utm_source=chatgpt.com "A PDF to Markdown converter - GitHub"
[5]: https://www.reddit.com/r/MachineLearning/comments/1hg5d3p/p_vision_parse_parse_pdf_documents_into_markdown/ "[P] Vision Parse: Parse PDF documents into Markdown formatted content using Vision LLMs : r/MachineLearning"
[6]: https://github.com/kermitt2/grobid?utm_source=chatgpt.com "kermitt2/grobid: A machine learning software for extracting ... - GitHub"
[7]: https://pandoc.org/MANUAL.html?utm_source=chatgpt.com "Pandoc User's Guide"
[8]: https://github.com/facebookresearch/nougat?utm_source=chatgpt.com "Implementation of Nougat Neural Optical Understanding for ... - GitHub"
[9]: https://github.com/vinaykanigicherla/pdftolatex "GitHub - vinaykanigicherla/pdftolatex: Python tool for generation of LaTex code from PDF files."
[10]: https://github.com/mcpeixoto/pdf2latex-converter?utm_source=chatgpt.com "mcpeixoto/pdf2latex-converter - GitHub"
[11]: https://github.com/aliang8/Paper2LaTeX "GitHub - aliang8/Paper2LaTeX: Transcribing handwritten graphs"
[12]: https://github.com/wzlxjtu/PDF2LaTeX-dataset?utm_source=chatgpt.com "wzlxjtu/PDF2LaTeX-dataset - GitHub"
[13]: https://github.com/NormXU/nougat-latex-ocr?utm_source=chatgpt.com "NormXU/nougat-latex-ocr: Codebase for fine-tuning ... - GitHub"
