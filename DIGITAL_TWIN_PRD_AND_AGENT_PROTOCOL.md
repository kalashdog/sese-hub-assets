# 🏭 Digital Twin 3D & Controle de Estoque (Dashboard 025)
## Documento de Contexto de Engenharia, Arquitetura e Protocolo de Agentes

> **Status:** Produção / Ativo (Versão `v2.0.0`)  
> **Filial Piloto:** Anchieta (Fábrica Volkswagen São Bernardo do Campo)  
> **Repositório Frontend:** `c:\Users\vinic\Desktop\VOLKSWAGEN\hubs-dashboards\anchieta\hub-dashboards`  
> **Repositório Assets 3D (CDN):** `https://github.com/kalashdog/sese-hub-assets`  
> **Arquivo Blender Master:** `OneDrive - Sese\...\025 - Ocupacao Estoque\armazem_anchieta_piloto.blend`  
> **Última Atualização:** 16/11/2026 – Nivelamento Global Z=0.00, Inversão Ala 14 e Pipeline de Publicação Automatizada.

---

## 1. Visão do Produto & Objetivo (PRD)

O **Digital Twin 3D** é a representação digital interativa da Fábrica Anchieta integrada ao SAP/WMS da Sesé. Ele transforma planilhas estáticas de ocupação em um mapa volumétrico 3D imersivo para tomada de decisão em tempo real:
1. **Visão Global (Macro):** Navegação aérea por toda a planta fabril, visualizando taxas de ocupação volumétricas e badges 2D/3D em tempo real para todas as alas (Alas 20, 14, 05, 11, 01, 02, 03, 04, 10, 13, 19, 22).
2. **Visão Detalhada de Galpão (Micro):** Isolamento dinâmico de galpão (*Culling* de alta performance). As demais alas são ocultadas para garantir 60 FPS, revelando corredores, marcações de piso, porta-paletes e cargas instanciadas em alta fidelidade.
3. **Inspeção de Palete & Estoque:** Clique ou hover sobre qualquer palete no espaço 3D para revelar Part Number SAP, descrição da peça, status de qualidade (`BLOQUEADO_Q`, `DISPONIVEL`, `LIVRE`) e quantidade, abrindo modal com visual glassmorphism.
4. **Navegação First-Person / Drone:** Física inspirada na Unreal Engine 5 (`WASD` + `Shift` para acelerar + `Botão Direito do Mouse` para orientar a câmera) e tecla de atalho **`F`** (*Focus*) para alinhar a câmera frontalmente no corredor do palete selecionado.

---

## 2. Arquitetura do Sistema & Stack Tecnológica

### 2.1. Frontend (`hub-dashboards`)
A base foi totalmente modularizada, eliminando o monólito anterior do laboratório:

```text
src/components/digitaltwin/
├── fabrica/                             # 📁 Regras de Negócio e Coordenadas por Galpão
│   ├── index.ts                         # Registry central (getAlaConfig, getPresetsParaAla)
│   ├── types.ts                         # Interfaces IAlaConfig, IWingStats, IPresetTarget
│   ├── global.ts                        # Presets e volumes da visão macro da fábrica
│   └── alas/
│       ├── ala20.ts                     # Ala 20 (Armazém Principal - 20 Ruas ortogonais)
│       └── ala14.ts                     # Ala 14 (Recepção & Cofre - Corredores perpendiculares)
│
├── composables/                         # ⚙️ Motores Especializados Neutros (Single Responsibility)
│   ├── useThreeScene.ts                 # Câmera Three.js, Renderer WebGL, Iluminação PBR e Resize
│   ├── useNavigationControls.ts         # Controles WASD + RMB, Clamping polar zenith e aceleração
│   ├── useCameraTransitions.ts          # Animações de voo GSAP e Vetor Universal de Picking ('F')
│   ├── useWarehouseModel.ts             # Download CDN, Cache API, Culling por Ala e Materiais
│   ├── usePalletInstancing.ts           # InstancedMesh, Modos Visuais (Realista, Ocupação, Qualidade)
│   ├── useRaycastInteraction.ts         # Raycasting GPU, Badges 2D, Tooltips e Seleção
│   ├── useWarehouseData.ts              # Agregação de dados, KPIs e Filtros
│   └── useSapPositionResolver.ts        # Parser de strings e chaves de-para do SAP
│
├── TopBar.vue                           # Header executivo: Botão Hub, KPIs e Modos Visuais
├── SideFilterPanel.vue                  # Painel retrátil: Alas, Ruas, Níveis e Filtros
├── DigitalTwinViewer.vue                # Viewport Three.js limpo e orquestrado
├── PositionDetailModal.vue              # Modal Glassmorphism de inspeção de material
└── DigitalTwinContainer.vue             # Container orquestrador do Dashboard 025
```

### 2.2. Pipeline de Modelos 3D & CDN Híbrida
* **Hospedagem Principal:** `kalashdog/sese-hub-assets` (GitHub público via `raw.githubusercontent.com`).
* **Cache Inteligente (`Cache API`):** Implementado em `useWarehouseModel.ts` sob a chave `digitaltwin-3d-cache-v2`. Na 1ª visita, o modelo (13.9 MB) é baixado com barra de progresso; nas visitas subsequentes, é lido do disco local em <200ms com **zero tráfego de rede**.
* **Fallback Automático:** Caso o usuário esteja em rede restrita que bloqueie o GitHub, o motor Three.js faz fallback automático para `/models/planta_anchieta_digitaltwin.glb?v=2.0.0` local.

---

## 3. Padrões Geométricos e Regras de Coordenadas (CRÍTICO)

> [!WARNING]
> Agentes que operarem no Blender ou no Three.js **DEVEM** respeitar estritamente as regras de eixos e centros geométricos descritas abaixo. Violar estas regras desalinha paletes do SAP e quebra a renderização.

### 3.1. Mapeamento de Eixos: Blender vs. Three.js
* **Blender (Espaço de Modelagem):**
  * $+X$ = Leste (Direita) | $-X$ = Oeste (Esquerda)
  * $+Y$ = Norte (Rua Principal / `RUA_DUAS_MAOS_ALA11_ALA14`) | $-Y$ = Sul
  * $+Z$ = Altura / Elevação
* **Three.js (Espaço do Navegador após exportação GLTF):**
  * $\text{Three.js } X = \text{Blender } X$
  * $\text{Three.js } Y = \text{Blender } Z \quad\text{(Altura)}$
  * $\text{Three.js } Z = -\text{Blender } Y \quad\text{(Profundidade invertida)}$

### 3.2. Regra Absoluta de Elevações (Cotas Z)
Todos os pisos operacionais principais da fábrica Anchieta estão **nivelados estritamente na cota $Z = 0.00\text{ m}$**:
* Piso **Ala 20**: $Z = 0.00\text{ m}$ (Racks em $Z = 0.02\text{ m}$).
* Laje Superior **Ala 11** (`Laje_Piso_ALA_11_N1`): $Z = 0.00\text{ m}$.
* Piso Logístico **Ala 05**: $Z = 0.00\text{ m}$.
* Piso Operacional **Ala 14** (`Piso_Logistico_ALA_14`): $Z = 0.00\text{ m}$ (Racks em $Z = 0.01\text{ m}$).
* Rua de Ligação (`RUA_DUAS_MAOS_ALA11_ALA14`): $Z = 0.01\text{ m}$ (Conectada perfeitamente entre Ala 11 e Ala 14).
* **Níveis Inferiores (Térreo/Subsolo):**
  * Nível 0 (Térreo das alas de 3 andares): $Z = -10.00\text{ m}$.
  * Nível -1 (Subsolo): $Z = -20.00\text{ m}$.
  * Pilares estruturais: Devem estender de $Z = -20.00\text{ m}$ a $Z = +8.00\text{ m}$.

### 3.3. Geometria de Porta-Paletes (Racks) e Inversão
* **Materiais de Orientação:**
  * **Frente:** Contém `Mat_EtiquetaPosicao` (etiquetas de código WMS) e `Mat_ProtetorAmarelo` (protetor de impacto de empilhadeira). **Deve sempre apontar para o corredor de circulação.**
  * **Traseira:** Contém `Mat_LimitadorTraseiro` (barra de parada que impede queda da carga).
* **Ponto de Origem (*Pivot Point*):**
  * Racks possuem origem no canto inferior de um dos pés de apoio (`Vector(1.65, 0.51, 4.30)` em espaço local).
  * **PROIBIDO** girar o rack apenas com `o.rotation_euler.z += math.pi`. Isso desloca a peça em ~2.7m para fora das faixas do chão.
  * **Fórmula Obrigatória de Rotação ao Redor do Centro:**
    ```python
    center_orig = sum([r.matrix_world @ Vector(c) for c in r.bound_box], Vector((0,0,0))) / 8.0
    center_local = sum([Vector(c) for c in r.bound_box], Vector((0,0,0))) / 8.0
    scaled_center_local = Vector((center_local.x * r.scale.x, center_local.y * r.scale.y, center_local.z * r.scale.z))
    new_rot_z = orig_rot.z + math.pi
    R_new = Euler((orig_rot.x, orig_rot.y, new_rot_z), 'XYZ').to_matrix().to_3x3()
    r.rotation_euler.z = new_rot_z
    r.location = center_orig - R_new @ scaled_center_local
    ```

---

## 4. Protocolo de Colaboração Multi-Agente (Regras de Conduta)

> [!IMPORTANT]
> Qualquer agente de IA ou desenvolvedor humano que atuar neste projeto deve seguir rigorosamente as 6 regras abaixo.

### Regra 1: Registro Obrigatório de Alterações (Changelog / Claiming)
* Antes de iniciar um trabalho, consulte a **Seção 5 (Registro de Alterações)** para verificar o que foi entregue e garantir que você não sobreponha tarefas em andamento.
* Ao finalizar a sua alteração, **adicione obrigatoriamente** uma nova linha no Registro com data, autor, escopo, arquivos modificados e justificativa.

### Regra 2: Preservação dos 75 Dashboards Existentes
* O Hub possui 75 dashboards operando na mesma aplicação Vue (`DashboardView.vue`). O Digital Twin é o ID `025`.
* Nunca altere estilos globais (`style.css`), layouts de header ou classes de containers sem escopo (`scoped`).
* Use sempre guardas condicionais (`v-if="dashboard.id === '025'"` ou `!== '025'`) para comportamentos específicos do 3D.

### Regra 3: Checagem Rigorosa de Tipagem (`vue-tsc`)
* Após qualquer alteração no frontend, execute o type-check estrito:
  ```powershell
  cd "c:\Users\vinic\Desktop\VOLKSWAGEN\hubs-dashboards\anchieta\hub-dashboards"
  npx vue-tsc --build --noEmit
  ```
* Nenhuma tarefa é considerada "concluída" se `vue-tsc` retornar erros.

### Regra 4: Segurança em Operações do Blender
* Sempre crie um backup antes de rodar scripts em lote:
  `shutil.copyfile(filepath, filepath.replace('.blend', '_backup_nome.blend'))`
* Inspecione o estado antes de salvar (`min_z`, `max_z`, `rot_z`).
* Salve a cena com `bpy.ops.wm.save_mainfile()`.

### Regra 5: Publicação Automatizada de Modelos
* Ao atualizar o `.blend`, **nunca** faça upload manual no navegador.
* Use o pipeline automatizado via terminal:
  ```powershell
  npm run publish:3d
  ```
* Ou execute o script interno salvo no `.blend`: `export_and_publish_glb.py`.
* **Sempre incremente a versão do cache** em `useWarehouseModel.ts` (`digitaltwin-3d-cache-vX`) para invalidar o cache dos navegadores dos usuários.

### Regra 6: Filosofia ETL (Regras de Ouro Sesé)
* Caso atue em scripts de dados/ETL: estritamente **Polars (lazy) + Calamine + DuckDB**.
* **Proibido:** `pandas`, `openpyxl`, `numpy`.
* Nunca use códigos de filial como números inteiros; mantenha sempre como `Utf8`.

---

## 5. Registro de Alterações dos Agentes (Changelog de Engenharia)

> **Instrução para novos agentes:** Insira sua entrada no topo desta tabela ao concluir sua entrega.

| Versão / Data | Autor / Agente | Escopo / Módulos Afetados | Resumo Técnico da Alteração | Status de Validação |
| :--- | :--- | :--- | :--- | :---: |
| **v2.0.0**<br>16/11/2026 | **Antigravity Lead Architect** | `armazem_anchieta_piloto.blend`<br>`useWarehouseModel.ts`<br>`ala14.ts`<br>`sese-hub-assets` | **Nivelamento Global Z=0.00 & Inversão Ala 14:**<br>1. Invertidos 775 racks e 1.034 paletes da Ala 14 em 180° mantendo centro milimétrico.<br>2. Rebaixados 2.302 objetos da Ala 14 em -10.0m nivelando com Ala 20, 11 e 05.<br>3. Criado pipeline automatizado `npm run publish:3d` e script interno no Blender.<br>4. Incrementado Cache API para `digitaltwin-3d-cache-v2`. | **Aprovado & Ao Vivo**<br>(GitHub Commit `218e934`) |
| **v1.1.0**<br>15/11/2026 | **Antigravity Lead Architect** | `useWarehouseModel.ts`<br>`kalashdog/sese-hub-assets` | **Arquitetura CDN Híbrida & Cache API:**<br>Implementado download com progresso de `raw.githubusercontent.com` com armazenamento na `Cache API` do navegador (0 bytes de rede em reloads) e fallback offline local. | **Aprovado & Testado**<br>(E2E Playwright OK) |
| **v1.0.0**<br>14/11/2026 | **Antigravity Lead Architect** | `src/components/digitaltwin/*` | **Modularização do Monólito & Paridade:**<br>Migração do PoC `lab-digital-twin` para `hub-dashboards` modularizado (9 composables + subpastas por Ala), resolução de colisão de CSS, atalho 'F' e paridade visual 100%. | **Aprovado & Testado**<br>(`vue-tsc` 0 erros) |

---

## 6. Backlog & Roteiro de Tarefas para Próximos Agentes

1. **Expansão de Alas Restantes:**
   - **Ala 05 (Estamparia Norte):** Mapear ruas e presets detalhados em `src/components/digitaltwin/fabrica/alas/ala05.ts`.
   - **Ala 11 (Montagem & Qualidade):** Mapear os 2 níveis (N1 = 0m, N0 = -10m) e corredores longitudinais.
   - **Ala 04 (Subconjuntos):** Mapear mezanino e pulmão.
2. **Otimização de Geometria:**
   - Implementar *instancing* em malhas de racks vazios para reduzir polígonos renderizados na GPU.
3. **Novas Features no Frontend:**
   - Filtro avançado por Família de Peça e Fornecedor direto no `SideFilterPanel.vue`.
   - Visualização de mapa de calor (*Heatmap*) de rotatividade / giro de estoque (Fast/Slow movers).
