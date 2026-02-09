# =============================================================================
# PATCH PTIWorkbookBuilder — Substituir _b_versions e _b_diagram
# =============================================================================
#
# INSTRUÇÕES:
#   1. No seu app.py, localize o método `def _b_versions(self):` dentro da
#      classe PTIWorkbookBuilder e SUBSTITUA-O inteiramente pelo código abaixo.
#   2. Faça o mesmo com `def _b_diagram(self):`.
#   3. Não altere nenhum outro método da classe.
#
# DIAGNÓSTICO DOS PROBLEMAS CORRIGIDOS:
#
# ┌─────────────────────────────────────────────────────────────────────────┐
# │ ABA VERSÕES                                                            │
# ├─────────────────────────────────────────────────────────────────────────┤
# │ BUG 1: CNs e Áreas Locais vinham APENAS de vivo_rows. Se a Engenharia │
# │        não preenchia a tabela Dados VIVO, colunas CN e ÁREAS ficavam   │
# │        vazias. CORREÇÃO: fallback para op_rows.                        │
# │                                                                        │
# │ BUG 2: A coluna "Áreas Locais" não incluía UF. Agora formata como     │
# │        "São Paulo/SP, Campinas/SP".                                    │
# │                                                                        │
# │ BUG 3: Colunas 4-10 (Data, Resp Eng, Resp Atacado, Escopo, CN, Áreas) │
# │        usavam um dicionário `fd` que mapeava colunas por número, mas   │
# │        as colunas de dado real (CN=8, Áreas=9) dependiam de variáveis  │
# │        que poderiam estar vazias. CORREÇÃO: escrita explícita célula   │
# │        por célula com fallbacks robustos.                              │
# ├─────────────────────────────────────────────────────────────────────────┤
# │ ABA DIAGRAMA DE INTERLIGAÇÃO                                           │
# ├─────────────────────────────────────────────────────────────────────────┤
# │ BUG 1: Tabelas Ponta A/B tinham apenas 3 colunas (Tráfego, End IP,    │
# │        Mask) e não mapeavam SBC, Localidade, Cidade/UF do formulário.  │
# │        CORREÇÃO: 6 colunas por ponta com todos os campos.             │
# │                                                                        │
# │ BUG 2: Múltiplos blocos (um por CN/vivo_row) colidiram verticalmente — │
# │        todos escreviam a partir da linha 15 fixa.                      │
# │        CORREÇÃO: offset vertical calculado por bloco.                  │
# │                                                                        │
# │ BUG 3: Dados da operadora (SBC, faixa_ip, endereço) nunca eram        │
# │        escritos nas células. CORREÇÃO: mapeamento completo.            │
# │                                                                        │
# │ BUG 4: Nenhuma seção de "Detalhes do Enlace" existia para exibir      │
# │        coordenadas, EOT LC/LD, referência. CORREÇÃO: sub-tabela.      │
# │                                                                        │
# │ BUG 5: Se imagem não existia, lançava FileNotFoundError e crashava.   │
# │        CORREÇÃO: exibe mensagem de aviso na célula.                    │
# └─────────────────────────────────────────────────────────────────────────┘
#
# =============================================================================


    # -----------------------------------------------------------------
    # MÉTODO 1: _b_versions  (substituir o existente na classe)
    # -----------------------------------------------------------------
    def _b_versions(self):
        """
        Aba 'Versões' — Controle de versões do PTI.
        Preenche TODAS as colunas com dados do formulário.
        """
        s = self.s
        ws = self.wb.create_sheet(title="Versões")

        # Larguras
        self._cw(ws, {
            1: 2, 2: 2, 3: 8, 4: 12, 5: 28, 6: 28,
            7: 28, 8: 10, 9: 18, 10: 10, 11: 2
        })

        # Header + subtítulo
        self._bh(ws, 2, 3, 10)
        self._st(ws, 4, 3, 10, "CONTROLE DE VERSÕES DO PTI")

        # Cabeçalhos da tabela (linha 6)
        headers = [
            "Versão", "Data", "Responsável Eng de ITX",
            "Responsável Gestão de ITX", "Escopo", "CN",
            "ÁREAS LOCAIS", "ATA"
        ]
        for i, t in enumerate(headers):
            c = ws.cell(row=6, column=3 + i, value=t)
            c.font = Font(name="Calibri", size=9, bold=True, color="FFFFFF")
            c.alignment = s.align_center
            c.fill = s.fill_light
        ws.row_dimensions[6].height = 25.0

        # ── Coleta inteligente de dados ──────────────────────────────
        # CNs: prioriza vivo_rows, fallback para op_rows
        all_cns = self.cns_unicos[:]
        if not all_cns:
            all_cns = self._unique_field(self.op_rows, "cn")

        # Áreas Locais: prioriza vivo_rows, fallback para op_rows
        all_areas = self.areas_locais[:]
        if not all_areas:
            all_areas = self._unique_field(self.op_rows, "cidade")

        # Formata áreas com UF: "São Paulo/SP, Campinas/SP"
        areas_com_uf = []
        for area in all_areas:
            uf_match = ""
            for row in (self.vivo_rows + self.op_rows):
                if row.get("cidade", "").strip() == area and row.get("uf", "").strip():
                    uf_match = row["uf"].strip()
                    break
            areas_com_uf.append(f"{area}/{uf_match}" if uf_match else area)

        cns_text = ", ".join(all_cns) if all_cns else ""
        areas_text = ", ".join(areas_com_uf) if areas_com_uf else ""

        # ── Preenche as 10 linhas ────────────────────────────────────
        for i in range(10):
            r = 7 + i
            ws.row_dimensions[r].height = 22.0
            f = s.alt_fill(i)

            # Col 3: Versão
            ws.cell(row=r, column=3, value=i + 1).font = s.font_body
            ws.cell(row=r, column=3).fill = f
            ws.cell(row=r, column=3).alignment = s.align_center

            if i == 0:
                # Primeira linha: todos os dados preenchidos
                cell_data = {
                    4:  (self.created_str,  s.align_center),   # Data
                    5:  (self.resp_eng,     s.align_left),     # Resp Eng ITX
                    6:  (self.resp_atk,     s.align_left),     # Resp Gestão ITX
                    7:  (self.escopo_text,  s.align_wrap),     # Escopo
                    8:  (cns_text,          s.align_center),   # CN
                    9:  (areas_text,        s.align_wrap),     # Áreas Locais
                    10: ("",                s.align_center),   # ATA
                }
                for col, (value, align) in cell_data.items():
                    c = ws.cell(row=r, column=col, value=value)
                    c.font = s.font_body
                    c.fill = f
                    c.alignment = align
            else:
                # Demais linhas: vazias mas formatadas
                for col in range(4, 11):
                    c = ws.cell(row=r, column=col, value="")
                    c.font = s.font_body
                    c.fill = f
                    c.alignment = (
                        s.align_wrap if col in (7, 9)
                        else (s.align_left if col in (5, 6) else s.align_center)
                    )

        ws.freeze_panes = "C7"
        self._ps(ws, ls=True)


    # -----------------------------------------------------------------
    # MÉTODO 2: _b_diagram  (substituir o existente na classe)
    # -----------------------------------------------------------------
    def _b_diagram(self):
        """
        Aba 'Diagrama de Interligação' — Um bloco completo por CN/vivo_row.

        Cada bloco contém:
        - Header: CN + Localidade (Cidade/UF) + Ref/Data/Escopo
        - Tabela Ponta A (VIVO): Tráfego, SBC, End IP, Mask, Local, Cidade/UF
        - Tabela Ponta B (Oper): Tráfego, SBC, End IP, Faixa IP, Local, Cidade/UF
        - Detalhes do Enlace: SBC, endereços, máscara, coordenadas, EOT
        - Imagem do diagrama (ao final de todos os blocos)
        """
        s = self.s
        ws = self.wb.create_sheet(title="Diagrama de Interligação")
        cfg = self._diagram_cfg()
        n = self.nome

        # ── Layout de colunas ────────────────────────────────────────
        # 1-2: margem | 3-8: Ponta A | 9-14: Ponta B | 15: margem
        self._cw(ws, {
            1: 2, 2: 2,
            3: 14, 4: 14, 5: 18, 6: 14, 7: 16, 8: 14,
            9: 14, 10: 14, 11: 18, 12: 14, 13: 16, 14: 14,
            15: 2
        })

        # ── Cabeçalho global ─────────────────────────────────────────
        self._bh(ws, 2, 3, 14)

        global_texts = [
            (4,  f"Diagrama de Interligação entre a VIVO e a {n}", True),
            (6,  "2.2.1 DIAGRAMAÇÃO DO PROJETO SIP", True),
            (7,  "2.2.1.1 Anúncio de Redes pelos 2 Links SPC", True),
            (8,  f"A {n} abordará os endereços da VIVO conforme abaixo.", False),
            (9,  f"A VIVO abordará os endereços da {n} conforme abaixo.", False),
            (11, "2.2.1.2 Parâmetros de Configuração do Link", True),
        ]
        for row_n, text, bold in global_texts:
            ws.merge_cells(start_row=row_n, start_column=3,
                           end_row=row_n, end_column=14)
            c = ws.cell(row=row_n, column=3, value=text)
            c.font = Font(name="Calibri", size=11 if bold else 10, bold=bold)
            c.alignment = s.align_left if bold else s.align_wrap
            ws.row_dimensions[row_n].height = 22.0

        # ASN (global)
        for offset, label, value in [
            (0, "ASN VIVO", "10429 (Público)"),
            (1, f"ASN {n}", self.asn_op),
        ]:
            r = 12 + offset
            ws.merge_cells(start_row=r, start_column=3, end_row=r, end_column=5)
            ws.cell(row=r, column=3, value=label).font = Font(
                name="Calibri", size=10, bold=True)
            ws.cell(row=r, column=3).alignment = s.align_center
            ws.merge_cells(start_row=r, start_column=6, end_row=r, end_column=8)
            ws.cell(row=r, column=6, value=value).font = s.font_body
            ws.cell(row=r, column=6).alignment = s.align_center

        # ── Itens de tráfego (escopo) ────────────────────────────────
        traffic_items = [t for t in self.traffic if t]
        if not traffic_items:
            traffic_items = [""]
        num_traffic_rows = max(len(traffic_items), 3)

        # Número de blocos
        num_blocks = max(1, len(self.vivo_rows))

        # Altura de cada bloco: header(1) + ponta(1) + thead(1) + tráfego(N) +
        #                       gap(1) + detail_header(1) + detail_rows(5) + gap(2) = N+12
        BLOCK_HEIGHT = num_traffic_rows + 12
        block_start_row = 15

        # ═════════════════════════════════════════════════════════════
        # BLOCOS (um por vivo_row / CN)
        # ═════════════════════════════════════════════════════════════
        for b_idx in range(num_blocks):
            vivo_row = self.vivo_rows[b_idx] if b_idx < len(self.vivo_rows) else {}
            op_row = self._find_matching_op_row(vivo_row.get("cn", ""), b_idx) or {}

            br = block_start_row + (b_idx * BLOCK_HEIGHT)

            # ── Dados do formulário ──────────────────────────────
            cn_val          = vivo_row.get("cn", "")
            ref_val         = vivo_row.get("ref", "")
            data_val        = vivo_row.get("data", "")
            escopo_val      = vivo_row.get("escopo", "")
            localidade_vivo = vivo_row.get("localidade", "")
            sbc_vivo        = vivo_row.get("sbc", "")
            mask_val        = vivo_row.get("mask", "")
            endereco_vivo   = vivo_row.get("endereco_link", "")
            cidade_vivo     = vivo_row.get("cidade", "")
            uf_vivo         = vivo_row.get("uf", "")
            lat_vivo        = vivo_row.get("lat", "")
            long_vivo       = vivo_row.get("long", "")

            localidade_op   = op_row.get("localidade", "")
            sbc_op          = op_row.get("sbc", "")
            faixa_ip_op     = op_row.get("faixa_ip", "")
            endereco_op     = op_row.get("endereco_link", "")
            cidade_op       = op_row.get("cidade", "")
            uf_op           = op_row.get("uf", "")
            eto_lc          = op_row.get("eto_lc", "")
            eot_ld          = op_row.get("eot_ld", "")

            # Cidade/UF display (com fallback)
            cidade_d = cidade_vivo or cidade_op or ""
            uf_d     = uf_vivo or uf_op or ""
            loc_tag  = f" — {cidade_d}/{uf_d}" if cidade_d and uf_d else (
                       f" — {cidade_d}" if cidade_d else "")

            # ── ROW 0: Header do CN ──────────────────────────────
            ws.merge_cells(start_row=br, start_column=3,
                           end_row=br, end_column=8)
            c = ws.cell(row=br, column=3,
                        value=f"CN {cn_val}{loc_tag}" if cn_val
                        else f"Bloco {b_idx + 1}")
            c.font = Font(name="Calibri", size=10, bold=True, color="FFFFFF")
            c.alignment = s.align_center
            c.fill = s.fill_light
            ws.row_dimensions[br].height = 25.0

            # Ref / Data / Escopo ao lado
            ws.merge_cells(start_row=br, start_column=9,
                           end_row=br, end_column=14)
            info_parts = []
            if ref_val:    info_parts.append(f"Ref: {ref_val}")
            if data_val:   info_parts.append(f"Data: {data_val}")
            if escopo_val: info_parts.append(f"Escopo: {escopo_val}")
            c2 = ws.cell(row=br, column=9,
                         value="  |  ".join(info_parts) if info_parts
                         else "VRF: _______________")
            c2.font = Font(name="Calibri", size=9, bold=True, color="FFFFFF")
            c2.alignment = s.align_center
            c2.fill = s.fill_light

            # ── ROW 1: Ponta A / Ponta B ─────────────────────────
            ponta_row = br + 1
            ws.row_dimensions[ponta_row].height = 22.0
            for sc, ec, text in [
                (3, 8, "Ponta A — VIVO"),
                (9, 14, f"Ponta B — {n}"),
            ]:
                ws.merge_cells(start_row=ponta_row, start_column=sc,
                               end_row=ponta_row, end_column=ec)
                c = ws.cell(row=ponta_row, column=sc, value=text)
                c.font = Font(name="Calibri", size=10, bold=True, color="FFFFFF")
                c.alignment = s.align_center
                c.fill = s.fill_secondary

            # ── ROW 2: Cabeçalhos da tabela ──────────────────────
            thead_row = br + 2
            ws.row_dimensions[thead_row].height = 25.0

            ponta_a_headers = [
                (3, "Tráfego"), (4, "SBC"), (5, "Endereço IP"),
                (6, "NET MASK"), (7, "Localidade"), (8, "Cidade/UF"),
            ]
            ponta_b_headers = [
                (9, "Tráfego"), (10, "SBC"), (11, "Endereço IP"),
                (12, "Faixa IP"), (13, "Localidade"), (14, "Cidade/UF"),
            ]
            for col, h in (ponta_a_headers + ponta_b_headers):
                c = ws.cell(row=thead_row, column=col, value=h)
                c.font = s.font_subheader
                c.alignment = s.align_center
                c.fill = s.fill_accent
                c.border = s.box_border

            # ── ROWS 3+: Dados de tráfego ────────────────────────
            # Formato Cidade/UF
            def _fmt_cidade_uf(cid, uf):
                if cid and uf:
                    return f"{cid}/{uf}"
                return cid or ""

            cidade_uf_vivo = _fmt_cidade_uf(cidade_vivo, uf_vivo)
            cidade_uf_op   = _fmt_cidade_uf(cidade_op, uf_op)

            for j in range(num_traffic_rows):
                dr = thead_row + 1 + j
                ws.row_dimensions[dr].height = 22.0
                fl = s.alt_fill(j)
                traf = traffic_items[j] if j < len(traffic_items) else ""
                first = (j == 0)  # Dados fixos só na 1ª linha

                # Ponta A (VIVO): 6 colunas
                row_a = [
                    (3,  traf),                                   # Tráfego
                    (4,  sbc_vivo if first else ""),              # SBC
                    (5,  endereco_vivo if first else ""),         # End IP
                    (6,  mask_val if first else ""),              # Mask
                    (7,  localidade_vivo if first else ""),       # Localidade
                    (8,  cidade_uf_vivo if first else ""),        # Cidade/UF
                ]
                # Ponta B (Operadora): 6 colunas
                row_b = [
                    (9,  traf),                                   # Tráfego
                    (10, sbc_op if first else ""),                # SBC
                    (11, endereco_op if first else ""),           # End IP
                    (12, faixa_ip_op if first else ""),           # Faixa IP
                    (13, localidade_op if first else ""),         # Localidade
                    (14, cidade_uf_op if first else ""),          # Cidade/UF
                ]

                for col, val in (row_a + row_b):
                    c = ws.cell(row=dr, column=col, value=val)
                    c.font = s.font_small
                    c.fill = fl
                    c.border = s.box_border
                    c.alignment = s.align_center

            # ── DETALHES DO ENLACE ───────────────────────────────
            detail_header_row = thead_row + 1 + num_traffic_rows + 1

            ws.merge_cells(start_row=detail_header_row, start_column=3,
                           end_row=detail_header_row, end_column=14)
            c = ws.cell(row=detail_header_row, column=3,
                        value=f"Detalhes do Enlace — CN {cn_val}" if cn_val
                        else f"Detalhes do Enlace — Bloco {b_idx + 1}")
            c.font = Font(name="Calibri", size=9, bold=True, color="FFFFFF")
            c.alignment = s.align_center
            c.fill = s.fill_secondary
            ws.row_dimensions[detail_header_row].height = 20.0

            details = [
                ("SBC VIVO:",       sbc_vivo,
                 f"SBC {n}:",       sbc_op),
                ("Endereço Link:",  endereco_vivo,
                 "Endereço Link:",  endereco_op),
                ("Máscara:",        mask_val,
                 "Faixa IP:",       faixa_ip_op),
                ("Localidade:",     localidade_vivo,
                 "Localidade:",     localidade_op),
                ("Coordenadas:",    f"{lat_vivo}, {long_vivo}" if lat_vivo else "",
                 "EOT LC / LD:",    f"{eto_lc} / {eot_ld}" if (eto_lc or eot_ld) else ""),
            ]

            for d_idx, (lab_a, val_a, lab_b, val_b) in enumerate(details):
                dr = detail_header_row + 1 + d_idx
                ws.row_dimensions[dr].height = 18.0
                fl = s.alt_fill(d_idx)

                # Ponta A: label (3-4) + valor (5-8)
                ws.merge_cells(start_row=dr, start_column=3,
                               end_row=dr, end_column=4)
                c = ws.cell(row=dr, column=3, value=lab_a)
                c.font = Font(name="Calibri", size=8, bold=True)
                c.alignment = s.align_left
                c.fill = fl
                c.border = s.box_border

                ws.merge_cells(start_row=dr, start_column=5,
                               end_row=dr, end_column=8)
                c = ws.cell(row=dr, column=5, value=val_a)
                c.font = s.font_small
                c.alignment = s.align_left
                c.fill = fl
                c.border = s.box_border

                # Ponta B: label (9-10) + valor (11-14)
                ws.merge_cells(start_row=dr, start_column=9,
                               end_row=dr, end_column=10)
                c = ws.cell(row=dr, column=9, value=lab_b)
                c.font = Font(name="Calibri", size=8, bold=True)
                c.alignment = s.align_left
                c.fill = fl
                c.border = s.box_border

                ws.merge_cells(start_row=dr, start_column=11,
                               end_row=dr, end_column=14)
                c = ws.cell(row=dr, column=11, value=val_b)
                c.font = s.font_small
                c.alignment = s.align_left
                c.fill = fl
                c.border = s.box_border

        # ═════════════════════════════════════════════════════════════
        # IMAGEM DO DIAGRAMA (abaixo de todos os blocos)
        # ═════════════════════════════════════════════════════════════
        img_row = block_start_row + (num_blocks * BLOCK_HEIGHT) + 2

        if not os.path.exists(cfg["img"]):
            # Imagem não encontrada — aviso em vez de crash
            ws.merge_cells(start_row=img_row, start_column=3,
                           end_row=img_row, end_column=14)
            ws.cell(row=img_row, column=3,
                    value=f"⚠ Imagem não encontrada: {cfg['img']}"
            ).font = Font(name="Calibri", size=10, color="CC0000")
        else:
            el = [
                {"text": "Roteador OP", "xy_pct": cfg["r1"],
                 "font_size_pct": cfg["rfp"], "stroke_width_pct": cfg["rsp"]},
                {"text": "Roteador OP", "xy_pct": cfg["r2"],
                 "font_size_pct": cfg["rfp"], "stroke_width_pct": cfg["rsp"]},
                {"text": "Link resp Vivo", "xy_pct": cfg["lvxy"],
                 "font_size_pct": cfg["lfsz"], "stroke_width_pct": 0.006},
                {"text": f"Link resp {n}", "xy_pct": cfg["loxy"],
                 "font_size_pct": cfg["lfsz"], "stroke_width_pct": 0.006},
                {"text": "RAC/RAV/HL4", "xy_pct": (0.33, 0.70),
                 "font_size_pct": 0.010, "stroke_width_pct": 0.006},
                {"text": "RAC/RAV/HL4", "xy_pct": (0.34, 0.60),
                 "font_size_pct": 0.010, "stroke_width_pct": 0.006},
            ]
            try:
                ann = render_labels_on_image(
                    image_path=cfg["img"], vivo_text="VIVO",
                    operator_text=n, vivo_xy_pct=cfg["vxy"],
                    operator_xy_pct=cfg["oxy"], font_path=cfg["fp"],
                    font_size_pct=cfg["fsz"], stroke_width_pct=0.009,
                    extra_labels=el,
                )
                xi = XLImage(ann)
                try:
                    with PILImage.open(ann) as src:
                        ow, oh = src.size
                except Exception:
                    ow, oh = 800, 300
                fw, fh = fit_size_keep_aspect(ow, oh, cfg["bw"], cfg["bh"])
                xi.width, xi.height = fw, fh
                anchor_col = get_column_letter(3 + cfg["aco"])
                anchor_row = max(1, img_row + cfg["aro"])
                ws.add_image(xi, f"{anchor_col}{anchor_row}")
            except Exception as e:
                logger.error(f"Erro ao renderizar imagem do diagrama: {e}")
                ws.merge_cells(start_row=img_row, start_column=3,
                               end_row=img_row, end_column=14)
                ws.cell(row=img_row, column=3,
                        value=f"⚠ Erro ao gerar imagem: {e}"
                ).font = Font(name="Calibri", size=10, color="CC0000")

        self._ps(ws, ls=True)
