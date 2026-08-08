# Lý thuyết
- Lựa chọn encoder:
	- PhoBERT: [VinAIResearch/PhoBERT: PhoBERT: Pre-trained language models for Vietnamese (EMNLP-2020 Findings)](https://github.com/VinAIResearch/PhoBERT)
	- Pipeline PhoBert.
	- PhoBERT trích xuất embedding.
	- Cosine Similarity: cos(u, v) = uv / (|u|.|v|)
	- Bi-Encoder: Dữ liệu riêng nhau hết.
	- Cross-Encoder: Tài liệu chung tài liệu kia.
	- Kiến trúc SBERT.
		- Ứng dụng: Semantic similary, semantic search, FAQ matching, clustering, duplicate detection, recomendation, RAG.
		- - Multilingual-E5:
	- E5: họ embedding model được thiết kế mạnh cho retrieval. Một đặc điểm quan trọng là sử dụng prefix để mô hình nhận biết vai trò văn bản.
	- Phù hợp: E5 phù hợp semantic search, cross-lugual retrival, document retrieval và RAG tiếng Việt mà không cần tự huấn luyện từ đầu.
	- BGE-M3 - Embedding đa chức năng
		- BGE-M3 hỗ trợ 3 kiểu biểu diễn chính:
			- Dense retrieval: một dense vector đại diện toàn văn bản
			- Sparse retrieval: giữ tín hiệu texical/ keyword
			- Multi-vector retrieval: nhiều vector/ token-level interaction.
		- Dense mạnh về paraphrase và ngữ nghĩa, nhưng sparse thường tốt hơn với tên riêng, mã văn bản, mã học phần, số hiệu hoặc keyword chính xác.
		- Do đó: Retrieval hiện đại thường đi theo hướng semantic + lexical ...
	- Chọn mô hình nào:

| **Bài toán**             | **Mô hình phù hợp**      | **Ghi chú**     |
| ------------------------ | ------------------------ | --------------- |
| NER/POS tiếng Việt       | PhoBERT                  | Token-level     |
| Text classification      | PhoBERT / encoder VN     | Fine-tuning     |
| Sentence similarity      | SBERT                    | Sentence-level  |
| Semantic search          | E5 / BGE-M3              | Retrieval       |
| RAG tiếng Việt           | E5 / BGE-M3              | Dense retrieval |
| Cross-lingual search     | multilingual-E5 / BGE-M3 | Đa ngôn ngữ     |
| Exact keyword + semantic | BGE-M3 hoặc Dense + BM25 | Hybrid          |
| Final ranking            | Cross-Encoder / Reranker | Top-K only      |

- Contrastive learning: Tìm tại liệu nào đúng hay sai (Tài liệu dương/ Tài liệu âm).
- Query phân loại:
	- Positive: Sinh viên được xét toosit nghiệp khi tích lũy đủ số tín chỉ
	- Easy negative: Dự báo thời tiết Cần Thơ ngày mai.
	- Hard negative: Điều kiện đăng ký học phần ngành Khoa học Máy Tính.
	- -> Negative có thể lấy từ BM25, embedding gần nhất, in-batch negatives hoặc sinh bằng LLM.

- Semantic search:
	- Pipeline: Query -> Embedding -> Similarity Search -> Top-K.
- Vector DB & FAISS:
	- Đánh index: Faiss.index
	- Approximate Nearest Neighbor (ANN): Đánh đổi tốc độ search.
	
> Không hỏi "mô hình nào mạnh nhất?", mà hỏi "mô hình nào phù hợp nhất với task và evaluation protocol?"
> Retrival quality là một bottleneck của RAG.

- Kết hợp bộ encoder.
- Không nên đưa một tài liệu 20 trang thành một embedding duy nhất. Thay vào đó chia $D \to \{c_1, c_2, \dots, c_n\}$ rồi embed từng chunk.
- Chiến lược
	- Fixed-length.
	- Sentence chunking.
	- Paragraph chunking.
	- Recursive chunking.
	- Semantic chunking.
	- Hierarchical chunking.
- Traffoff
	- Chunk quá ngắn: mất context.
	- Chunk quá dài: nhiều nội dung nhiễu.
	- Overlap quá lớn: database phình to.
	- Overlap quá nhỏ: dễ cắt mất ý.

> Embedding model mạnh nhưng chunking kém vẫn có thể tạo retrieval/RAG kém.

- Nén LLM chuyên gia, xử lý song song.
- Pipeline retrieval hiện đại:
	- `Query` → [`Dense Embedding` & `BM25 Sparse`] → `Fusion` → `Top-100` → `Cross-Encoder Reranker` → `Top-5`
	- Một pipeline mạnh thường là **Dense Retrieval + Sparse Retrieval + Fusion + Reranking**.
	- **Dense**: giúp hiểu paraphrase/ngữ nghĩa.
	- **BM25**: giữ exact lexical matches.
	- **Reranker**: đánh giá sâu query-document trên tập ứng viên nhỏ.
	
- Fine-tune Endding.
- Đánh giá Embedding.
	- Semantic similarity
		- Pearson correlation
		- Spearman correlation
	- Retrieval
		- - Hit@K / Recall@K.
		- Precision@K.
		- MRR@K.
		- nDCG@K
	- 2 chỉ số đánh giá LLM khác:
		- **Bleu**
		- **Rough**
	- Công thức MRR
	
> Embedding dùng cho RAG phải được đánh giá ở retrieval level, không chỉ đánh giá câu trả lời cuối của LLM.

- PhoBERT vs SBERT vs E5 vs BGE-M3.

| **Model**         | **Cấp biểu diễn**      | **Điểm mạnh**        | **Điểm yếu**           | **Ứng dụng**           |
| ----------------- | ---------------------- | -------------------- | ---------------------- | ---------------------- |
| **PhoBERT**       | Token/context          | Tiếng Việt mạnh      | Không tối ưu retrieval | NER, classification    |
| **SBERT**         | Sentence               | Similarity tốt       | Tùy backbone/data      | Similarity, clustering |
| **E5**            | Sentence/document      | Retrieval mạnh       | Cần đúng prefix        | Search, RAG            |
| **BGE-M3**        | Dense + sparse + multi | Hybrid, multilingual | Nặng hơn               | Advanced retrieval/RAG |
| **Cross-Encoder** | Pair                   | Relevance chính xác  | Chậm                   | Reranking              |

> PhoBERT là **language encoder**; E5/BGE là **retrieval-oriented embedding models**. Hai nhóm giải quyết những mục tiêu không hoàn toàn giống nhau.

- ANI vs AGI.
- Sparse retrieval: BM25.
- Hybrid retrieval.
- Query rewriting.
- Multi-Query retrieval.
- Context compression.
- RAG failure modes.
- Lemma and Sense.
- Đồng nghĩa and Tương đồng.
- Hyponymy and Hypernymy.
- Entailment: quan hệ kéo theo.
- Biểu diễn từ phân tán.
	- Brown Cluster.
	- Word2Vec: CBOW vs Skip-gram.
	- Bài tập: tạo mẫu huấn luyện CBOW.
	
- GloVe: Global Vectors.
- Ma trận đồng xuất hiện.
- FastText: subword and OOV: nhanh và chính xác cao - sai chính tả, bắt lỗi chính tả.
- BPE and WordPiece: tách từ thành các đơn vị con phổ biến trong Transformer.
- Ứng dụng: kiểm tra sai chính tả
- TFIDF

- Các kỹ thuật của NER: có tính ứng dụng cao.
	- Glinner.
- POS tagging.
	
- Sai lầm thường gặp:
	- Lấy trực tiếp vector [CLS] của BERT/PhoBERT rồi mặc định đó là sentence embedding tối ưu.
	- Không normalize vector nhưng dùng inner product như cosine.
	- Dùng một embedding model cho mọi domain/task.
	- Không benchmark trên dữ liệu tiếng Việt thực tế.
	- Fine-tune retrieval nhưng chỉ dùng easy negatives.
	- Chỉ dùng dense retrieval, bỏ qua keyword/sparse retrieval.
	- Chunk quá dài hoặc quá ngắn.
	- Không đánh giá retriever riêng khi đánh giá RAG.
	- Không rerank khi candidate set có nhiều hard negatives.
	
- Tài nguyên từ vựng trong NLP:
	- WordNet: mạng từ vựng gồm synset và quan hệ ngữ nghĩa.
	- SentiWorldNet: mở rộng WordNet với điểm cảm xúc.
	- VerbNet: Mạng động từ, mô tả quan hệ cú pháp - ngữ nghĩa.
	- FrameNet: mô tả khung ngữ nghĩa của sự kiện và vai tham gia.

- Chủ đề:
	- Tôi có một hệ thống muốn hỗ trợ cho người nông dân, đang gặp một sự cố không biết làm sao trồng được sầu riêng: ứ quá nhiều -> bán không được -> nghe đồn xuất khẩu không được, điều tra đo do đất bị nhiễm chất hóa học tăng nhiều. Với kỹ thuật NLP, có thể làm được gì.
	- Góc nhìn: Extract thông tin của đất, kiểm tra thông tin của mẫu đất -> hệ thống upload kết quả đất -> recommend thông tin. Kho dữ liệu đất, ...
	- Thông tin trên mạng gôm về, phân nhóm, phân loại, xử lý thông tin liên quan đến pháp lý.
	- Chatbot cho cộng đồng hợp tác xã.
	- Đầu vào là text: xuất phát từ rất nhiều nguồn sách, video, audio. Tính chất: Sequence to sequence.
	- Text không phải là text: nhiệt độ, gen, ký hiệu, tấm hình, âm thanh, điện tâm đồ, quang phổ.
	- Frame hình động tác thể dụng: mã hóa chuỗi hình ảnh động tác -> đưa về bài toán LLM.

### Thực hành
- Tìm kiếm: SBert huggingface
- [RAG.ipynb - Colab](https://colab.research.google.com/drive/1CVlTp8rSZWjtRkxLEBRkikwZum-guC7A?usp=sharing)
- [finetune_qwen_vietnamese_colab-TrainingLLM.ipynb - Google Drive](https://drive.google.com/file/d/1m81zu5BfFhYtdvbCXN9l1LTw3gfMqB7c/view)
