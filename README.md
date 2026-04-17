if os.path.exists(cfg["img"]):
                cursor += 4
                img_anchor_row = cursor
                sbc_op_img = (op_row.get("sbc", "") if op_row else "") or ""
                row_labels = [
                    {"text": f"ROTEADOR {n}", "xy_pct": (0.65, 0.36), "font_size_pct": cfg["rfp"], "stroke_width_pct": cfg["rsp"]},
                    {"text": f"ROTEADOR {n}", "xy_pct": (0.65, 0.55), "font_size_pct": cfg["rfp"], "stroke_width_pct": cfg["rsp"]},
                    {"text": "LINK RESP VIVO", "xy_pct": (0.50, 0.70), "font_size_pct": cfg["lfsz"], "stroke_width_pct": 0.006},
                    {"text": f"LINK RESP {n}", "xy_pct": (0.50, 0.18), "font_size_pct": cfg["lfsz"], "stroke_width_pct": 0.006},
                    {"text": "ROTEADOR VIVO", "xy_pct": (0.34, 0.36), "font_size_pct": 0.010, "stroke_width_pct": 0.006},
                    {"text": "ROTEADOR VIVO", "xy_pct": (0.34, 0.55), "font_size_pct": 0.010, "stroke_width_pct": 0.006},
                    {"text": "CDSIP_SPO_PL", "xy_pct": (0.14, 0.41), "font_size_pct": 0.022, "stroke_width_pct": 0.006},
                    {"text": "CDSIP_SPO_JG", "xy_pct": (0.14, 0.58), "font_size_pct": 0.022, "stroke_width_pct": 0.006},
                    # Títulos acima das nuvens (não mais dentro delas)
                    {"text": "VIVO", "xy_pct": (0.14, 0.15), "font_size_pct": 0.035, "stroke_width_pct": 0.008},
                    {"text": n,      "xy_pct": (0.87, 0.15), "font_size_pct": 0.035, "stroke_width_pct": 0.008},
                ]
                if pl_block:
                    row_labels.append({"text": pl_block, "xy_pct": (0.14, 0.45), "font_size_pct": 0.020, "stroke_width_pct": 0.004})
                if jg_block:
                    row_labels.append({"text": jg_block, "xy_pct": (0.14, 0.62), "font_size_pct": 0.020, "stroke_width_pct": 0.004})
                # IP da rede reservada IPAM
                if ip_op_base:
                    row_labels.append({"text": ip_op_base, "xy_pct": (0.87, 0.45), "font_size_pct": 0.020, "stroke_width_pct": 0.005})
                # SBC da operadora (preenchido no formulário Atacado → Dados Operadora)
                if sbc_op_img:
                    row_labels.append({"text": sbc_op_img, "xy_pct": (0.87, 0.41), "font_size_pct": 0.022, "stroke_width_pct": 0.006})
                if cn_val:
                    cn_img_text = f"CN {cn_val}" + (f" — {cn_cidade}/{cn_uf}" if cn_cidade else "")
                    row_labels.append({"text": cn_img_text, "xy_pct": (0.50, 0.08), "font_size_pct": 0.028, "stroke_width_pct": 0.007})
                try:
                    ann = render_labels_on_image(
                        image_path=cfg["img"], vivo_text="", operator_text="",
                        vivo_xy_pct=cfg["vxy"], operator_xy_pct=cfg["oxy"],
                        font_path=cfg["fp"], font_size_pct=cfg["fsz"],
                        stroke_width_pct=0.009, extra_labels=row_labels,
                    )
                    xi = XLImage(ann)
                    try:
                        with PILImage.open(ann) as src:
                            ow, oh = src.size
                    except Exception:
                        ow, oh = 800, 300
                    fw, fh = fit_size_keep_aspect(ow, oh, cfg["bw"], cfg["bh"])
                    xi.width, xi.height = fw, fh
                    anchor_col = get_column_letter(LC + cfg["aco"])
                    anchor_row = max(1, img_anchor_row + cfg["aro"])
                    ws.add_image(xi, f"{anchor_col}{anchor_row}")
                    cursor = img_anchor_row + max(22, int(fh / 15) + 2) + 2
                except Exception as exc:
                    logger.warning("Erro ao gerar imagem (bloco %d): %s", b_idx + 1, exc)
                    cursor += 3
            else:
                if b_idx == 0:
                    logger.warning("Imagem base não encontrada: %s", cfg["img"])
                cursor += 3
        self._ps(ws, ls=True)
