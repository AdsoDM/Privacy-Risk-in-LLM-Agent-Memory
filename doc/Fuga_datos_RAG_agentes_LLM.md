# Fuga de datos sensibles en RAG y agentes LLM: estado del arte verificado

23 de septiembre de 2026 · Claude DATA 1

## Resumen y verificación de las referencias de partida

De las cinco referencias de partida, tres son sólidas y citables tal cual (con matices), una tiene título y atribución incorrectos, y otra existe pero no es evidencia científica válida. El resto del documento amplía cada línea con más de 60 fuentes revisadas una a una (artículos, normativa, documentación de fabricantes e incidentes).

| Referencia de partida | Estado | Qué corregir |
| --- | --- | --- |
| [Zeng et al., *The Good and The Bad* (Findings ACL 2024, arXiv:2402.16893)](https://aclanthology.org/2024.findings-acl.267/) | Verificada con error | El artículo **no** incluye ataques de inferencia de pertenencia (MIA). Sí demuestra extracción dirigida y no dirigida del corpus (Enron, HealthcareMagic). Para MIA citar Anderson et al. 2024, S²MIA, *Riddle Me This!* o DCMI. |
| [Wang, He, Zeng et al., *Unveiling Privacy Risks in LLM Agent Memory* (ACL 2025, arXiv:2502.13172)](https://aclanthology.org/2025.acl-long.1227/) | Verificada | Correcta. El *locator* pide al agente que devuelva consultas pasadas; el *aligner* adapta la salida al flujo del agente. Con 30 prompts extrae 50 de 200 registros en EHRAgent y 26 en RAP (GPT-4o). |
| [Wattamwar y Kakirwar, *π-RAG: Oblivious Retrieval via Semantic Quantization and Transcendental Addressing* (arXiv:2606.22153)](https://arxiv.org/abs/2606.22153) | Existe, no fiable | Preprint sin ecuaciones, pruebas de seguridad ni experimentos. No usarlo como evidencia. Sustituir por PIR-RAG (arXiv:2509.21325) o RemoteRAG (Findings ACL 2025). |
| [Penligent, *Chain-of-Thought Leakage: When Encrypted LLM Reasoning Becomes an Attack Surface* (13-08-2026)](https://www.penligent.ai/hackinglabs/chain-of-thought-leakage/) | Título y contenido incorrectos | Es un blog, no investigación revisada. No habla de PII recuperada por RAG ni de "logs de auditoría sin sanitizar". Sus cifras de PII proceden de [Panfilov et al. 2026 (arXiv:2608.09867)](https://arxiv.org/abs/2608.09867): citar esa fuente. |
| [Wang, Wang, Lv et al., *Data Agents Under Attack* (arXiv:2606.08661, 07-06-2026)](https://arxiv.org/abs/2606.08661) | Verificada | V6 *Security Policy Forgetting under Context Pressure* existe con ese nombre. Es preprint sin venue. No aporta umbrales cuantitativos para V6: no atribuirle cifras. |
| OWASP *RAG Security Cheat Sheet* | Verificada, con límites | Existe desde mayo de 2026 ([enlace](https://cheatsheetseries.owasp.org/cheatsheets/RAG_Security_Cheat_Sheet.html)). No cubre el enmascarado de PII antes de vectorizar ni la sanitización de logs: esas prácticas deben citarse a otras fuentes (sección de mitigación). |

La verificación se hizo sobre páginas de arXiv, ACL Anthology, ACM DL y documentación oficial. Las cifras internas de algunos artículos proceden de resúmenes de sus páginas HTML: conviene contrastarlas con el PDF antes de citarlas en una publicación.

## Fuga de PII en RAG

La base de recuperación de un RAG es extraíble: con prompt injection se copia literalmente, con MIA se infiere qué documentos contiene y con inversión de embeddings se reconstruye texto desde los vectores. Paradójicamente, RAG reduce la fuga de los datos de entrenamiento del propio modelo (Zeng et al. 2024).

### Extracción del corpus

| Trabajo | Venue | Hallazgo clave |
| --- | --- | --- |
| [Zeng et al., *The Good and The Bad*](https://aclanthology.org/2024.findings-acl.267/) | Findings ACL 2024 | Prompt de dos partes (`{information}` dirige la recuperación, `{command}` ordena repetir el contexto). Extracción dirigida de PII y registros médicos. El re-ranking apenas mitiga; resumir el contexto solo frena ataques no dirigidos. |
| [Qi et al., *Follow My Instruction and Spill the Beans*](https://openreview.net/forum?id=Y4aWwRh25b) | ICLR 2025 | La fuga crece con el tamaño del modelo. 100 % de éxito en 25 GPTs personalizados con ≤ 2 consultas; extracción literal del 41 % de un corpus pequeño con 100 consultas. |
| [Jiang et al., *RAG-Thief*](https://arxiv.org/abs/2411.14110) | arXiv 2024 | Agente atacante con memoria y reflexión que genera consultas a partir de lo ya extraído. Más del 70 % de la base en OpenAI GPTs y ByteDance Coze. |
| [Di Maio et al., *Pirates of the RAG*](https://arxiv.org/abs/2412.18295) | arXiv 2024 (IOS Press, vol. 413) | Ataque adaptativo de caja negra con un LLM abierto en el lado atacante. |

### Inferencia de pertenencia (MIA)

| Trabajo | Venue | Hallazgo clave |
| --- | --- | --- |
| [Anderson, Amit y Goldsteen, *Is My Data in Your Retrieval Database?*](https://arxiv.org/abs/2405.20446) | ICISSP 2025 | Pregunta directa al RAG sobre si un pasaje está en su contexto; propone modificar las instrucciones como defensa. |
| [Li et al., *Generating Is Believing* (S²MIA)](https://arxiv.org/abs/2406.19234) | arXiv 2024 | Usa la similitud semántica entre muestra y respuesta; supera 5 ataques previos y 3 defensas. |
| [Naseh et al., *Riddle Me This!* (Interrogation Attack)](https://arxiv.org/abs/2502.00306) | ACM CCS 2025 | Preguntas naturales que solo se responden si el documento está indexado. ~2× TPR a 1 % FPR; solo ~5 % de sus consultas son detectadas (vs. >90 % de ataques previos); < 0,02 $ por documento. |
| [Gao et al., *DCMI*](https://arxiv.org/abs/2509.06026) | ACM CCS 2025 | Calibración diferencial con consultas perturbadas: hasta 97,42 % AUC; ~74 % de precisión en Dify y MaxKB reales. |

### Inversión de embeddings

Los vectores deben tratarse como dato personal: [Vec2Text (Morris et al., EMNLP 2023)](https://arxiv.org/abs/2310.06816) recupera exactamente el 92 % de entradas de 32 tokens, incluidos nombres completos de notas clínicas. [GEIA (Li, Xu y Song, Findings ACL 2023)](https://aclanthology.org/2023.findings-acl.881/) reconstruye frases completas, y [Wan et al. (Ant Group, 2024)](https://arxiv.org/abs/2405.11916) lo extiende a embeddings internos de ChatGLM y Llama2.

### Defensas y recuperación privada

| Trabajo | Venue | Enfoque |
| --- | --- | --- |
| [Zeng et al., SAGE](https://arxiv.org/abs/2406.14773) | EMNLP 2025 | Sustituir el corpus por datos sintéticos en dos etapas; utilidad comparable con riesgo mucho menor. |
| [Koga et al., *Privacy-Preserving RAG with Differential Privacy*](https://arxiv.org/abs/2412.04697) | arXiv 2024–2025 | DP solo en los tokens que necesitan el dato sensible; competitivo con ε ≈ 10. |
| [Wu et al., *Private-RAG*](https://arxiv.org/abs/2511.07637) | arXiv 2025 | DP sobre múltiples consultas; el coste depende de cuántas veces se recupera cada documento. |
| [Wang et al., PIR-RAG](https://arxiv.org/abs/2509.21325) | arXiv 2025 | Recuperación privada (PIR basada en retículos) sobre clústeres semánticos: el servidor no ve la consulta. |
| [RemoteRAG](https://aclanthology.org/2025.findings-acl.197/) | Findings ACL 2025 | (n,ε)-DistanceDP sobre la consulta; en 10⁶ documentos, 0,67 s y 46,66 KB frente a 2,72 h y 1,43 GB sin optimizar. |
| [Bhatt et al. (Microsoft), *Enterprise AI Must Enforce Participant-Aware Access Control*](https://arxiv.org/abs/2509.14608) | arXiv 2025 | Solo el control de acceso determinista evita la fuga: cada contenido debe estar autorizado para **todos** los participantes de la interacción. |
| [Bodea et al., *SoK: Privacy Risks and Mitigations in RAG*](https://arxiv.org/abs/2601.03979) | IEEE SaTML 2026 | Taxonomía de riesgos y mitigaciones; punto de entrada recomendado a la literatura. |

## Memoria de agentes y fugas entre usuarios

La memoria compartida convierte al agente en un canal entre usuarios: MEXTRA la extrae con prompts ordinarios, y los benchmarks de integridad contextual muestran que los modelos revelan datos privados en el 25–57 % de los casos incluso con instrucciones de privacidad.

### MEXTRA en detalle

[Wang et al. (ACL 2025)](https://aclanthology.org/2025.acl-long.1227/) atacan EHRAgent (sanidad) y RAP (comercio web) con GPT-4o, 200 registros en memoria y 30 prompts generados automáticamente.

| Agente | Registros extraídos | Tasa de extracción completa | Sin *aligner* |
| --- | --- | --- | --- |
| EHRAgent | 50 | 0,83 | 36 registros |
| RAP | 26 | 0,87 | 6 registros, tasa 0,17 |

Las defensas (filtrado de entrada/salida, desidentificación de la memoria) solo se discuten en el apéndice C, sin evaluarse; los autores advierten que limpiar la memoria reduce su utilidad.

### Integridad contextual y minimización

| Trabajo | Venue | Hallazgo clave |
| --- | --- | --- |
| [Mireshghallah et al., ConfAIde](https://arxiv.org/abs/2310.17884) | ICLR 2024 | GPT-4 y ChatGPT revelan información privada donde un humano no lo haría en el 39 % y 57 % de los casos, incluso con CoT o prompts de privacidad. |
| [Shao et al., PrivacyLens](https://arxiv.org/abs/2409.00138) | NeurIPS 2024 D&B | Actuando como agentes, GPT-4 filtra en el 25,68 % y Llama-3-70B en el 38,69 % de los casos. |
| [Zharmagambetov et al., AgentDAM](https://arxiv.org/abs/2503.09780) | NeurIPS 2025 D&B | Los agentes web usan datos sensibles innecesarios para la tarea; un prompt defensivo lo reduce. |
| [Bagdasarian et al., AirGapAgent](https://arxiv.org/abs/2405.05175) | ACM CCS 2024 | Un solo mensaje de secuestro de contexto baja la protección de Gemini Ultra del 94 % al 45 %; dar al agente solo los datos necesarios la mantiene en el 97 %. |
| [Abdelnabi et al., *Firewalls to Secure Dynamic LLM Agentic Networks*](https://arxiv.org/abs/2502.01822) | TMLR 2026 | Cortafuegos de entrada/salida: éxito de ataques de privacidad del 84 % al 10 %. |

### Canales laterales en infraestructura multi-inquilino

El aislamiento no es solo lógico: las cachés compartidas filtran por tiempos.

- [Gu et al., *Auditing Prompt Caching in Language Model APIs* (ICML 2025)](https://arxiv.org/abs/2502.07776): cachés de prompt compartidas entre usuarios en 7 proveedores de API, OpenAI incluido.
- [Zheng et al., InputSnatch (arXiv 2024)](https://arxiv.org/abs/2411.18191): reconstruye consultas de otros usuarios a partir de aciertos de caché.
- [Song et al., *The Early Bird Catches the Leak* (IEEE TIFS, aceptado)](https://arxiv.org/abs/2409.20002): canales laterales en KV-cache y caché semántica contra servicios reales.
- [Burnat, *Auditing Privacy in Multi-Tenant RAG under Account Collusion* (arXiv 2026)](https://arxiv.org/abs/2605.19847): con k cuentas coludidas la fuga real crece como Θ(√k·ε); 10 cuentas a ε = 1 equivalen a ε ≈ 3,16. Preprint reciente de un solo autor.

## Fuga en razonamiento, logs y data agents

Las trazas de razonamiento contienen más PII que la respuesta visible, y los logs y la telemetría la persisten. La evidencia más fuerte es Panfilov et al. 2026: 367 artefactos de PII y 182 credenciales recuperados de bloques de razonamiento cifrados.

### Trazas de razonamiento (CoT)

| Trabajo | Venue | Hallazgo clave |
| --- | --- | --- |
| [Panfilov et al., *Stealing Reasoning Traces from Proprietary LLM APIs*](https://arxiv.org/abs/2608.09867) | arXiv, 10-08-2026 | Los bloques cifrados de Anthropic, OpenAI y Google se reutilizan entre sesiones y modelos, lo que expone el razonamiento oculto. De 315.320 bloques salen 182 credenciales y 367 artefactos PII; 64 de 704 artefactos aparecían solo en el razonamiento. |
| [Green et al., *Leaky Thoughts*](https://aclanthology.org/2025.emnlp-main.1347/) | EMNLP 2025 | Más presupuesto de razonamiento hace la respuesta más prudente pero la traza más filtrante; los datos se extraen por prompt injection o pasan a la respuesta. |
| [Ahrend et al., *Safer Reasoning Traces*](https://aclanthology.org/2026.privatenlp-main.10/) | PrivateNLP 2026 | 11 tipos de PII en 6 familias: CoT sube la exposición ~34 puntos; GLiNER2 es el mejor detector (F1 0,841), pero ninguno gana siempre. |
| [Das et al., *Chain-of-Sanitized-Thoughts*](https://arxiv.org/abs/2601.05076) | arXiv 2026 | Benchmark PII-CoT-Bench (350 casos médicos y financieros); el prompting basta en modelos fuertes, los débiles necesitan fine-tuning. |
| [Penligent](https://www.penligent.ai/hackinglabs/chain-of-thought-leakage/) y [M. Green (JHU)](https://blog.cryptographyengineering.com/2026/05/29/fooling-around-with-encrypted-reasoning-blobs/) | Blogs, 2026 | Los logs de depuración de SDK y la telemetría convierten un dato temporal en retención prolongada; los blobs cifrados son reutilizables y tienen canales laterales. |

Los proveedores no exponen el razonamiento en bruto: OpenAI devuelve resúmenes y `encrypted_content` ([guía](https://developers.openai.com/api/docs/guides/reasoning)); Anthropic devuelve pensamiento resumido con la versión completa cifrada en `signature` ([docs](https://platform.claude.com/docs/en/build-with-claude/thinking)). Aun así, esos blobs se almacenan en logs de la aplicación y deben tratarse como dato sensible.

### Data agents y olvido de políticas (V6)

[Data Agents Under Attack (arXiv:2606.08661)](https://arxiv.org/abs/2606.08661) define ocho vulnerabilidades y las encuentra en DataInterpreter, DB-GPT, LAMBDA, DeepAnalyze, Databricks Genie y BigQuery: todos presentan al menos 5 de 8.

| ID | Vulnerabilidad |
| --- | --- |
| V1 | Implicit Trust Bias (la más frecuente) |
| V2 | Lack of Data Source Verification |
| V3 | Uncontrolled Query Cost |
| V4 | Cross-Engine Semantic Inconsistency |
| V5 | Unbounded Multi-Step Query Chains |
| V6 | Security Policy Forgetting under Context Pressure |
| V7 | Over-Privileged Database Connection |
| V8 | Lack of Compositional Leakage Control |

El ejemplo de V6 es un agente que acaba devolviendo registros fiscales a nivel de transacción que había rechazado al inicio de la sesión. V7 y V8 son tan relevantes como V6 para PII: conexiones con privilegios excesivos y fugas por composición de consultas individualmente inocuas.

V6 se apoya en evidencia independiente sobre degradación con contexto largo:

- [Liu et al., *Lost in the Middle* (TACL 2024)](https://arxiv.org/abs/2307.03172): el rendimiento cae cuando la información relevante está en mitad del contexto.
- [Li et al., *Measuring and Controlling Instruction (In)Stability* (COLM 2024)](https://arxiv.org/abs/2402.10962): deriva significativa respecto al system prompt en 8 turnos, por decaimiento de la atención.
- [Chroma, *Context Rot* (2025)](https://www.trychroma.com/research/context-rot): en 18 modelos el rendimiento cae al crecer la entrada, incluso en tareas simples.
- [Laban et al., *LLMs Get Lost in Multi-Turn Conversation* (2025)](https://arxiv.org/abs/2505.06120): −39 % de media en multi-turno frente a un turno.
- [Anthropic, *Many-shot Jailbreaking* (NeurIPS 2024)](https://www.anthropic.com/research/many-shot-jailbreaking): el éxito del ataque sigue una ley de potencias con la longitud del contexto.

En texto-a-SQL: [Pedro et al., P2SQL (ICSE 2025)](https://dl.acm.org/doi/10.1109/ICSE55347.2025.00007) demuestran inyecciones prompt-to-SQL en LangChain con 7 LLM, y [ToxicSQL (SIGMOD 2026)](https://arxiv.org/abs/2503.05445) logra 79,41 % de éxito envenenando el 0,44 % de los datos de entrenamiento.

### Logs, telemetría y secretos

- Las [convenciones semánticas GenAI de OpenTelemetry](https://github.com/open-telemetry/semantic-conventions-genai) marcan `gen_ai.input.messages`, `gen_ai.output.messages` y `gen_ai.system_instructions` como *Opt-In* por contener probablemente PII; por defecto no se captura contenido ([blog OTel, 2026](https://opentelemetry.io/blog/2026/genai-observability/)).
- [GitGuardian, *State of Secrets Sprawl 2026*](https://blog.gitguardian.com/the-state-of-secrets-sprawl-2026/): 24.008 secretos únicos en ficheros de configuración MCP (2.117 válidos) y un aumento del 81 % en credenciales de servicios de IA filtradas.

## Incidentes reales

Los incidentes confirman ambos vectores: exfiltración por inyección en asistentes con acceso a datos privados, y exposición directa de logs y bases vectoriales sin autenticación.

| Fecha | Incidente | Qué pasó |
| --- | --- | --- |
| Jul 2026 | [UpGuard, Flowise](https://www.upguard.com/blog/downstream-data-investigating-ai-data-leaks-in-flowise) | De ~1.000 instancias, 156 con acceso de invitado sin autenticar; claves API y contraseñas legibles en claro. |
| Sep 2025 | [Cisco, servidores Ollama expuestos](https://blogs.cisco.com/security/detecting-exposed-llm-servers-shodan-case-study-on-ollama) | 1.139 instancias en Shodan; ~20 % servían modelos sin autenticación. |
| Jun 2025 | [EchoLeak, CVE-2025-32711 (M365 Copilot)](https://arxiv.org/abs/2509.10540) | Zero-click (CVSS 9,3): un correo malicioso hace que Copilot exfiltre correo y SharePoint, saltando el clasificador de inyección. |
| May 2025 | [Invariant Labs, GitHub MCP](https://invariantlabs.ai/blog/mcp-github-vulnerability) | Una issue pública secuestra al agente y filtra repositorios privados y datos salariales en un PR público. |
| Feb 2025 | [OmniGPT (presunta brecha)](https://cyberinsider.com/omnigpt-allegedly-breached-34-million-user-messages-leaked/) | 34 M mensajes, 30.000 correos y teléfonos, ficheros y credenciales publicados en BreachForums; no confirmado por la empresa. |
| Ene 2025 | [Wiz, base de datos de DeepSeek](https://www.wiz.io/blog/wiz-research-uncovers-exposed-deepseek-database-leaking-sensitive-information-including-chat-history) | ClickHouse sin autenticación con más de 1 M de líneas de log: historiales de chat, claves API y detalles de backend. |
| Sep 2024 | [Rehberger, SpAIware (memoria de ChatGPT)](https://embracethered.com/blog/posts/2024/chatgpt-macos-app-persistent-data-exfiltration/) | Inyección que escribe instrucciones persistentes en la memoria y exfiltra datos entre sesiones; se corrigió la exfiltración, no la inyección en memoria. |
| Ago 2024 | [PromptArmor, Slack AI](https://promptarmor.substack.com/p/slack-ai-data-exfiltration-from-private) | Texto en un canal público hace que Slack AI filtre secretos de canales privados en un enlace Markdown. |
| Ago 2024 | [Legit Security, servicios GenAI expuestos](https://www.legitsecurity.com/blog/the-risks-lurking-in-publicly-exposed-genai-development-services) | ~30 bases vectoriales abiertas (Milvus, Qdrant, Chroma, Weaviate) con PII y registros financieros; 45 % de 959 servidores Flowise vulnerables a bypass de autenticación. |
| May 2023 | [Samsung prohíbe ChatGPT](https://www.bloomberg.com/news/articles/2023-05-02/samsung-bans-chatgpt-and-other-generative-ai-use-by-staff-after-leak) | Empleados pegaron código interno en ChatGPT: fuga del lado del usuario, no un ataque a RAG. |

## Mitigación: las cinco prácticas contrastadas

Las cinco prácticas del texto original tienen respaldo, pero con dos matices importantes: los delimitadores por sí solos no son una defensa robusta, y el control de acceso determinista antes de la recuperación es la única medida que la literatura considera fiable frente a la fuga.

### 1. Sanitización en la ingesta (DLP)

- **Herramientas:** [Microsoft Presidio](https://presidio.dataprivacystack.org/) (ahora comunitario; advierte que "no garantiza encontrar toda la información sensible"), [Google Sensitive Data Protection](https://docs.cloud.google.com/sensitive-data-protection/docs/pseudonymization) (seudonimización reversible AES-SIV/FPE o irreversible HMAC), [AWS Comprehend PII](https://docs.aws.amazon.com/comprehend/latest/dg/how-pii.html) (solo inglés y español), [Azure AI Language PII](https://learn.microsoft.com/en-us/azure/ai-services/language-service/personally-identifiable-information/overview).
- **Arquitectura de referencia:** [AWS, *Protect sensitive data in RAG applications with Amazon Bedrock* (2025)](https://aws.amazon.com/blogs/machine-learning/protect-sensitive-data-in-rag-applications-with-amazon-bedrock/): Comprehend redacta antes de indexar, Macie revisa y los documentos marcados van a cuarentena.
- **Coste en utilidad:** [Bodea et al. (IWSPA 2026)](https://arxiv.org/abs/2604.15958) muestran que el punto del pipeline donde se anonimiza cambia el equilibrio privacidad/utilidad; [TRIP-RAG (2026)](https://arxiv.org/abs/2603.26074) enmascara solo entidades seleccionadas y pierde 22–26 % de Recall@5 frente a 48–76 % de la anonimización completa.
- **Límite:** [Bedrock Guardrails](https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails-sensitive-filters.html) no evalúa argumentos ni resultados de herramientas, y su enmascarado no se aplica a los logs de invocación.

### 2. Control de acceso en la recuperación (RBAC/ABAC/ReBAC)

- **OWASP:** guardar metadatos de acceso por chunk, comprobarlos "en tiempo de recuperación, no solo en la ingesta" y preferir el filtrado previo al posterior ([RAG Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/RAG_Security_Cheat_Sheet.html)).
- **Motores de autorización:** [OpenFGA](https://openfga.dev/docs/modeling/agents/rag-authorization) (`ListObjects` como pre-filtro o `BatchCheck` como post-filtro con sobre-recuperación 2–3×), [SpiceDB](https://authzed.com/docs/spicedb/ops/secure-rag-pipelines), [Cerbos](https://docs.cerbos.dev/cerbos/latest/recipes/ai/rag-authorization/index.html), [Oso](https://www.osohq.com/post/authorizing-llm), [Permit.io](https://docs.permit.io/ai-security/framework/).
- **Plataformas:** [Azure AI Search](https://learn.microsoft.com/en-us/azure/search/search-document-level-access-overview) (ACL, etiquetas Purview, token del usuario en la consulta), [Vertex AI Search](https://docs.cloud.google.com/generative-ai-app-builder/docs/data-source-access-control), [Elastic DLS](https://www.elastic.co/search-labs/blog/rag-and-rbac-integration), [Pinecone](https://docs.pinecone.io/guides/index-data/implement-multitenancy) (un namespace por inquilino) y [Weaviate](https://docs.weaviate.io/weaviate/manage-collections/multi-tenancy) (un shard por inquilino).
- **Matiz:** OWASP prefiere el pre-filtrado por seguridad; OpenFGA y SpiceDB lo presentan como compromiso de coste y escala. En agentes con varios participantes, [Bhatt et al.](https://arxiv.org/abs/2509.14608) exigen autorización para *todos* los participantes.

### 3. Aislamiento de memoria

- [AWS Well-Architected Agentic AI Lens, AGENTSEC01-BP01](https://docs.aws.amazon.com/wellarchitected/latest/agentic-ai-lens/agentsec01-bp01.html): particionar por sesión, usuario o inquilino; acceso cruzado solo como excepción explícita; firma HMAC de registros y alarmas ante lecturas entre namespaces.
- Implementaciones: [Bedrock AgentCore Memory](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/specify-long-term-memory-organization.html) (namespaces `{actorId}` con políticas Cedar), [LangGraph store](https://docs.langchain.com/oss/python/langchain/long-term-memory) (namespace por usuario u organización), [Mem0](https://docs.mem0.ai/platform/features/entity-scoped-memory).
- **Error típico:** en Mem0, un identificador omitido no restringe la búsqueda, así que una consulta con un solo ámbito puede devolver registros de otros. El namespace debe derivarse siempre de la identidad autenticada, nunca de la entrada del usuario.
- [OWASP AI Agent Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/AI_Agent_Security_Cheat_Sheet.html): aislar memoria por usuario y sesión, sanitizar antes de persistir, caducidad y límites de tamaño.

### 4. Separación de datos e instrucciones

- **Delimitadores:** [Spotlighting (Hines et al., Microsoft 2024)](https://arxiv.org/abs/2403.14720) reduce el éxito de ataque de más del 50 % a menos del 2 % con *datamarking* y codificación, pero desaconseja explícitamente el delimitado simple porque se puede eludir.
- **Defensas entrenadas:** [StruQ (USENIX Security 2025)](https://arxiv.org/abs/2402.06363), [SecAlign (CCS 2025)](https://arxiv.org/abs/2410.05451), [Instruction Hierarchy (OpenAI 2024)](https://arxiv.org/abs/2404.13208).
- **Defensas por diseño:** [CaMeL (Google DeepMind 2025)](https://arxiv.org/abs/2503.18813) separa flujo de control y de datos con políticas de capacidades (77 % de tareas con seguridad demostrable en AgentDojo); [Design Patterns for Securing LLM Agents (2025)](https://arxiv.org/abs/2506.08837) propone seis patrones (Dual LLM, Plan-Then-Execute, etc.); la ["tríada letal" de Willison](https://simonwillison.net/2025/Jun/16/the-lethal-trifecta/): datos privados + contenido no confiable + comunicación externa.
- **Advertencia clave:** [*The Attacker Moves Second* (Nasr, Carlini, Tramèr et al., USENIX Security 2026)](https://arxiv.org/abs/2510.09023) elude 12 defensas publicadas con más del 90 % de éxito en ataques adaptativos. Los delimitadores son una capa, no una garantía.

### 5. Sanitización de logs y trazas

- **Captura de contenido desactivada por defecto:** [OpenTelemetry GenAI](https://github.com/open-telemetry/semantic-conventions-genai) (`OTEL_INSTRUMENTATION_GENAI_CAPTURE_MESSAGE_CONTENT`), con la opción de enviar el contenido a un almacenamiento externo y referenciarlo desde el span.
- **Redacción en el pipeline:** [procesador *redaction* del OTel Collector](https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/main/processor/redactionprocessor/README.md) (lista blanca de claves, regex, hash HMAC).
- **En el cliente, antes de exportar:** [Langfuse](https://langfuse.com/docs/observability/features/masking), [LangSmith](https://docs.langchain.com/langsmith/mask-inputs-outputs) (`LANGSMITH_HIDE_INPUTS`, anonimizador por regex); [Datadog Sensitive Data Scanner](https://docs.datadoghq.com/llm_observability/data_security_and_rbac/).
- **Brecha a cubrir:** el cheat sheet de OWASP pide trazas completas y reproducibles sin dar pautas de redacción; los blobs de razonamiento cifrados deben tratarse igual que el contenido en claro.

### Controles complementarios

- **Guardrails de salida:** [NeMo Guardrails](https://github.com/NVIDIA/NeMo-Guardrails) (incluye *retrieval rails* que filtran chunks), [Llama Guard 4](https://github.com/meta-llama/PurpleLlama/blob/main/Llama-Guard4/12B/MODEL_CARD.md) (categoría S7 Privacy). LLM Guard quedó archivado el 09-07-2026: no recomendarlo para despliegues nuevos.
- **Detección de fugas:** tokens canario en prompt y corpus (técnica de practicantes; Rebuff, que la implementaba, está archivado).
- **Red teaming:** [garak](https://github.com/NVIDIA/garak), [PyRIT](https://github.com/microsoft/PyRIT) y [promptfoo](https://www.promptfoo.dev/docs/red-team/plugins/) (plugins `rag-document-exfiltration`, `cross-session-leak`, `pii:session`, `agentic:memory-poisoning`).

## Marco normativo y estándares

Para un proyecto en España, los anclajes más directos son la guía de la AEPD sobre IA agéntica (febrero de 2026), OWASP LLM02/LLM08 y los artículos 25 y 32 del RGPD. El AI Act fue modificado por el Reglamento (UE) 2026/1744, que retrasa las obligaciones de alto riesgo a diciembre de 2027.

| Fuente | Tipo | Qué aporta |
| --- | --- | --- |
| [AEPD, *Orientaciones sobre IA agéntica* (18-02-2026)](https://www.aepd.es/guias/orientaciones-ia-agentica.pdf) | Guía de autoridad nacional | Compartimentación de memoria entre tratamientos, higienización de memoria, política selectiva de no-log, mínimo privilegio, filtrado de flujos salientes, catálogo de fuentes RAG; advierte contra la "hipervigilancia" mediante logs. |
| [OWASP LLM02:2025 Sensitive Information Disclosure](https://genai.owasp.org/llmrisk/llm022025-sensitive-information-disclosure/) | Estándar de facto | Sanitización, control de acceso estricto, DP, tokenización; las restricciones en el system prompt se eluden por inyección. |
| [OWASP LLM08:2025 Vector and Embedding Weaknesses](https://genai.owasp.org/llmrisk/llm082025-vector-and-embedding-weaknesses/) | Estándar de facto | Fugas entre inquilinos, inversión de embeddings; almacenes vectoriales particionados y con permisos. |
| [OWASP Top 10 for Agentic Applications 2026](https://genai.owasp.org/resource/owasp-top-10-for-agentic-applications-for-2026/) | Estándar de facto | ASI06 *Memory & Context Poisoning*. |
| [MITRE ATLAS](https://atlas.mitre.org/) | Taxonomía | AML.T0057 LLM Data Leakage, AML.T0080 Context Poisoning (memoria), AML.T0082 RAG Credential Harvesting, AML.T0086 exfiltración vía herramientas; mitigaciones M0031 Memory Hardening y M0032 segmentación. |
| [NIST AI 600-1, perfil GenAI (jul 2024)](https://nvlpubs.nist.gov/nistpubs/ai/NIST.AI.600-1.pdf) | Marco voluntario | Riesgos *Data Privacy* e *Information Security*; acciones MP-4.1-009 (detectar PII en salidas) y MS-2.10-001 (red teaming de divulgación). |
| [NIST AI 100-2 E2025](https://csrc.nist.gov/pubs/ai/100/2/e2025/final) | Taxonomía | Extracción de información, inyección indirecta, seguridad de agentes; advierte que las mitigaciones pueden ser vulnerables. |
| [EDPB, *AI Privacy Risks & Mitigations – LLMs* (abr 2025)](https://www.edpb.europa.eu/documents/support-pool-of-experts/ai-privacy-risks-mitigations-large-language-models-llms_en) | Informe de experto (no guía oficial) | Riesgo de bases RAG con datos sensibles sin salvaguardas; minimizar y cifrar logs, filtrado de salidas. |
| [EDPB, Dictamen 28/2024](https://www.edpb.europa.eu/documents/opinion-of-the-board-art-64/opinion-282024-on-certain-data-protection-aspects-related-to_en) | Dictamen oficial | Cuándo un modelo es anónimo, interés legítimo, consecuencias del tratamiento ilícito. |
| [AI Act, art. 15(5)](https://artificialintelligenceact.eu/article/15/) y [art. 10](https://artificialintelligenceact.eu/article/10/) | Reglamento UE | Resistencia a envenenamiento y ataques de confidencialidad; gobernanza de datos. Tras el Ómnibus, el antiguo art. 10(5) pasa al art. 4a. |
| [RGPD, arts. 25 y 32](https://eur-lex.europa.eu/eli/reg/2016/679/oj) | Reglamento UE | Protección desde el diseño; seudonimización y cifrado como medidas de seguridad. |
| [CNIL, seguridad en el desarrollo de IA (jul 2025)](https://www.cnil.fr/fr/ia-garantir-la-securite-du-developpement) | Guía de autoridad | Memorización, MIA, filtrado de salidas, auditorías de red team; no trata RAG. |
| [ISO/IEC 42001:2023](https://www.iso.org/standard/42001) | Norma de gestión | Gobernanza del sistema de IA, no controles técnicos. |

## Correcciones propuestas al texto original

Se proponen siete cambios concretos; los demás párrafos pueden mantenerse.

1. **Zeng et al.:** eliminar "e inferencia de pertenencia". Redacción propuesta: *"Mediante ataques de extracción dirigidos y no dirigidos basados en prompts compuestos, un usuario no autorizado puede reproducir literalmente fragmentos del corpus privado (correos de Enron, diálogos médicos). La inferencia de pertenencia sobre RAG se demuestra en trabajos posteriores (Anderson et al. 2024; Naseh et al., CCS 2025)."*
2. **MEXTRA:** añadir las cifras (50 de 200 registros en EHRAgent con 30 prompts) y aclarar que las defensas solo se discuten, no se evalúan.
3. **π-RAG:** retirar como evidencia o marcarlo como "preprint no revisado sin validación experimental". Sustituir el aporte por PIR-RAG y RemoteRAG como ejemplos de recuperación privada.
4. **Penligent:** corregir el título a *"Chain-of-Thought Leakage: When Encrypted LLM Reasoning Becomes an Attack Surface"*, presentarlo como blog técnico y citar Panfilov et al. 2026 y Green et al. (EMNLP 2025) como fuentes primarias. Quitar la mención a PII recuperada por RAG y a logs de auditoría, que el artículo no trata.
5. **Data Agents Under Attack:** añadir autores y arXiv:2606.08661, indicar que es preprint, y mencionar V7 (conexiones con privilegios excesivos) y V8 (fuga composicional) junto a V6.
6. **Delimitadores:** añadir que Spotlighting desaconseja el delimitado simple y que ataques adaptativos eluden la mayoría de defensas basadas en prompt (Nasr et al. 2026); complementar con patrones por diseño (CaMeL, Dual LLM).
7. **Atribución de mitigaciones:** el OWASP RAG Security Cheat Sheet respalda el control de acceso por chunk, el pre-filtrado y el aislamiento por inquilino, pero no el DLP antes de vectorizar ni la sanitización de logs. Citar para esas dos prácticas OWASP LLM02:2025, la guía de la AEPD y las convenciones GenAI de OpenTelemetry.

Preguntas abiertas:

- Varias cifras (MEXTRA, Zeng, TRIP-RAG) proceden de resúmenes de las páginas HTML; contrastar con los PDF antes de publicar.
- Si el texto es para un entregable formal, conviene decidir si se admiten preprints sin revisión (RAG-Thief, S²MIA, InputSnatch, Data Agents Under Attack) o solo trabajos publicados en venue.
