# Html_StatusFaseOrdem4

Visual em HTML no Power BI para exibir o progresso por fase de cada ordem de produção.  
Utiliza DAX + HTML para mostrar:

- Fases planejadas (do General_report_Plm)
- Fases executadas (do BiEventsFull_PcFactory)
- Barra de progresso
- Cores por status (executado ou pendente)
- Tooltip com início e fim reais por fase

---

## 🔢 Requisitos de dados

Tabelas envolvidas:

- `General_report_Plm` com colunas:
  - `[Order No]`
  - `[Op Code]`
  - `[Op No]` (ordem das fases)

- `BiEventsFull_PcFactory` com colunas:
  - `[Ordem]`
  - `[Fase]`
  - `[ShiftDtStart]`
  - `[ShiftDtEnd]`

---

## 💡 DAX Completo

```dax
Html_StatusFaseOrdem4 = 
VAR Ordem = SELECTEDVALUE(BiEventsFull_PcFactory[Ordem])

VAR FasesPlanejadas =
    CALCULATETABLE(
        VALUES(General_report_Plm[Op Code]),
        General_report_Plm[Order No] = Ordem
    )

VAR FasesExecutadas =
    CALCULATETABLE(
        VALUES(BiEventsFull_PcFactory[Fase]),
        BiEventsFull_PcFactory[Ordem] = Ordem
    )

VAR QtdePlanejadas = COUNTROWS(FasesPlanejadas)
VAR QtdeExecutadas = COUNTROWS(
    INTERSECT(FasesPlanejadas, VALUES(BiEventsFull_PcFactory[Fase]))
)

VAR FasesOrdenadasComSeq =
    ADDCOLUMNS(
        FasesPlanejadas,
        "Seq",
            CALCULATE(
                MAX(General_report_Plm[Op No]),
                General_report_Plm[Order No] = Ordem &&
                General_report_Plm[Op Code] = EARLIER(General_report_Plm[Op Code])
            )
    )

VAR BarraFases =
    CONCATENATEX(
        FasesOrdenadasComSeq,
        VAR Fase = General_report_Plm[Op Code]
        VAR EstaExecutada = CONTAINS(VALUES(BiEventsFull_PcFactory[Fase]), BiEventsFull_PcFactory[Fase], Fase)

        VAR DataInicio =
            CALCULATE(
                MIN(BiEventsFull_PcFactory[ShiftDtStart]),
                BiEventsFull_PcFactory[Ordem] = Ordem &&
                BiEventsFull_PcFactory[Fase] = Fase
            )

        VAR DataFim =
            CALCULATE(
                MAX(BiEventsFull_PcFactory[ShiftDtEnd]),
                BiEventsFull_PcFactory[Ordem] = Ordem &&
                BiEventsFull_PcFactory[Fase] = Fase
            )

        VAR Tooltip =
            IF(
                EstaExecutada,
                "✅ Iniciado em " & FORMAT(DataInicio, "dd/MM/yyyy HH:mm") &
                "&#10;✅ Finalizado em " & FORMAT(DataFim, "dd/MM/yyyy HH:mm"),
                "⏳ Ainda não iniciado"
            )

        RETURN
            "<div title='" & Tooltip & "' style='padding:12px 24px; border-radius:12px; font-size:48px; font-weight:700; margin-right:12px; display:inline-block;
                background:" & IF(EstaExecutada, "#00B050", "#CCCCCC") & "; color:" & IF(EstaExecutada, "white", "#444") & ";'>
                " & Fase & "
            </div>",
        "",
        [Seq], ASC
    )

VAR Progresso = DIVIDE(QtdeExecutadas, QtdePlanejadas, 0)

VAR BarraVisual =
    "<div style='width:100%; height:54px; background:#ddd; border-radius:32px; overflow:hidden; margin-top:12px;'>
        <div style='width:" & FORMAT(Progresso, "0%") & "; height:54px; background:#00B050; transition: width 0.4s;'></div>
    </div>"

RETURN
"<div style='font-family:Segoe UI; font-size:45px; padding:24px; border:2px solid #ccc; border-radius:14px; background-color:#FAFAFA; box-shadow:inset 0 0 8px #ddd;'>
    <div style='margin-bottom:20px;'><strong>🆔 Ordem:</strong> " & Ordem & "</div>
    <div style='margin-bottom:14px;'><strong>🧩 Fases:</strong></div>
    <div style='margin:10px 0 20px 0;'>" & BarraFases & "</div>
    <div style='margin-bottom:14px;'><strong>📊 Progresso:</strong> " & FORMAT(Progresso, "0%") & "</div>
    " & BarraVisual & "
</div>"



