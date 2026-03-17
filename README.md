# Manual Técnico do Componente Calendário (D2Bridge) PIX PARA O CAFÉ: vitor.scarso@hotmail.com

Este pacote inclui 2 componentes visuais para Lazarus/Delphi que renderizam um calendário HTML (estilo “card” Bootstrap), com navegação e eventos vindos de um `TDataSet`:

- `TD2BridgeCalendario`: calendário mensal (1 mês por vez)
- `TD2BridgeCalendarioAno`: calendário em grade por meses (um intervalo de meses, geralmente 12)

O HTML gerado usa `{{CallBack=...}}` do D2Bridge para disparar ações (navegação, seleção de mês, seleção de dia).

---

## 1) Instalação (Lazarus / Delphi)

### Lazarus
1. Abra [calendario.lpk](./calendario.lpk).
2. Clique em **Compilar** e depois em **Instalar**.
3. Reinicie a IDE quando solicitado.
4. Os componentes aparecerão na paleta **VSComponents**.

### Delphi
1. Abra e instale os pacotes:
   - [Delphi/CalendarioD2BridgeR.dpk](./Delphi/CalendarioD2BridgeR.dpk)
   - [Delphi/CalendarioD2BridgeD.dpk](./Delphi/CalendarioD2BridgeD.dpk)
2. Recompile/instale e reinicie a IDE se necessário.

---

## 2) Requisitos do Front-end (Templates D2Bridge)

O HTML gerado assume que sua página já carrega (normalmente via template do D2Bridge):

- Bootstrap (classes como `card`, `table`, `btn`, `badge`, etc.)
- Font Awesome (ícones `fa fa-chevron-left/right`)

Se você usar `Appearance.Font = cfRoboto/cfOpenSans/cfLato`, o componente injeta automaticamente um `<link>` do Google Fonts no HTML gerado.

---

## 3) Dataset esperado (eventos)

O componente “lê” o `DataSet` no momento do `GenerateHTML` e agrega os registros por dia.

### Campos mínimos (recomendado)

Mapeie via `FieldMap` (Object Inspector):

- `FieldDate`: data do evento (o componente ignora a parte de hora, usando apenas a data)
- `FieldTitle`: texto do evento (se vazio, usa “Evento”)
- `FieldColor`: cor em hexadecimal (recomendado `#RRGGBB`), usada no modo de badges

`FieldID` existe no `FieldMap`, mas atualmente não é usado no HTML (reservado para evolução).

### Exemplo de SQL (genérico)

```sql
SELECT
  ID,
  DATA AS DATA,              -- DATE ou TIMESTAMP
  TITULO AS TITULO,          -- VARCHAR
  COR_EVENTO AS COR_EVENTO   -- ex: '#FF5733'
FROM EVENTOS
WHERE DATA BETWEEN :INI AND :FIM
ORDER BY DATA;
```

---

## 4) Renderização (o básico)

Você geralmente renderiza o calendário em um componente que aceita HTML (ex: `TLabel.Caption` em um `HTMLElement(Label)` do D2Bridge).

Exemplo (mensal):

```pascal
uses
  DateUtils;

procedure TForm1.AtualizaCalendario;
var
  Ini, Fim: TDateTime;
begin
  Ini := StartOfTheMonth(D2BridgeCalendario1.CurrentMonth);
  Fim := EndOfTheMonth(D2BridgeCalendario1.CurrentMonth);

  QueryEventos.Close;
  QueryEventos.ParamByName('INI').AsDate := Ini;
  QueryEventos.ParamByName('FIM').AsDate := Fim;
  QueryEventos.Open;

  LabelCalendario.Caption := D2BridgeCalendario1.GenerateHTML;
end;
```

---

## 5) CallBacks (navegação e seleção)

### Nomes de CallBack do calendário mensal (`TD2BridgeCalendario`)

O HTML dispara os seguintes callbacks:

- `CalendarMonthNav(prev|next|today)`
- `CalendarMonthSelect([this.value])` onde `this.value` é `yyyy-mm`
- `CalendarDaySelect(yyyy-mm-dd)`

Tratamento típico no seu `CallBack`:

```pascal
procedure TForm1.CallBack(const CallBackName: string; EventParams: TStrings);
begin
  inherited;

  if SameText(CallBackName, 'CalendarMonthNav') then
    D2BridgeCalendario1.DoInternalMonthNav(EventParams)
  else if SameText(CallBackName, 'CalendarMonthSelect') then
    D2BridgeCalendario1.DoInternalMonthSelect(EventParams)
  else if SameText(CallBackName, 'CalendarDaySelect') then
    D2BridgeCalendario1.DoInternalDaySelect(EventParams);

  AtualizaCalendario;
end;
```

Eventos expostos:

- `OnMonthChange(Sender, ISOYearMonth)` é disparado após `DoInternalMonthNav/DoInternalMonthSelect`
- `OnDayClick(Sender, ISODate)` é disparado por `DoInternalDaySelect`

### Nomes de CallBack do calendário por meses (`TD2BridgeCalendarioAno`)

- `CalendarYearNav(prev|next|today)` (anda em blocos de `MonthsToShow`)
- `CalendarYearMonthSelect(yyyy-mm)` (clique no nome do mês)
- `CalendarDaySelect(yyyy-mm-dd)`

Tratamento típico:

```pascal
procedure TForm1.CallBack(const CallBackName: string; EventParams: TStrings);
begin
  inherited;

  if SameText(CallBackName, 'CalendarYearNav') then
    D2BridgeCalendarioAno1.DoInternalYearNav(EventParams)
  else if SameText(CallBackName, 'CalendarYearMonthSelect') then
    D2BridgeCalendarioAno1.DoInternalYearMonthSelect(EventParams)
  else if SameText(CallBackName, 'CalendarDaySelect') then
    D2BridgeCalendarioAno1.DoInternalDaySelect(EventParams);

  AtualizaCalendarioAno;
end;
```

Eventos expostos:

- `OnMonthClick(Sender, ISOYearMonth)` é disparado por `DoInternalYearMonthSelect`
- `OnDayClick(Sender, ISODate)` é disparado por `DoInternalDaySelect`

---

## 6) Múltiplos calendários no mesmo HTML

Se você renderizar mais de um calendário no mesmo `Caption`/HTML, configure `CallbackPrefix` em cada instância:

- Calendário 1: `CallbackPrefix = 'CAL1_'`
- Calendário 2: `CallbackPrefix = 'CAL2_'`

Assim os callbacks ficam únicos:

- `CAL1_CalendarMonthNav`, `CAL1_CalendarDaySelect`, ...
- `CAL2_CalendarMonthNav`, `CAL2_CalendarDaySelect`, ...

Nesse caso, o seu `CallBackName` também chega com o prefixo, então o tratamento precisa considerar o nome completo.

---

## 7) Propriedades (Object Inspector)

### 7.1) Propriedades gerais — `TD2BridgeCalendario`

| Propriedade | Tipo | Padrão | Descrição |
| :--- | :--- | :--- | :--- |
| `DataSet` | `TDataSet` | `nil` | Fonte de dados com os eventos. |
| `FieldMap` | `TCalendarFieldMap` | - | Mapeamento dos campos do `DataSet` (veja 7.3). |
| `Appearance` | `TCalendarAppearance` | - | Configuração visual (veja 7.2). |
| `CurrentMonth` | `TDateTime` | `Date` | Mês atualmente exibido (o dia é ignorado). |
| `CallbackPrefix` | `string` | `''` | Prefixo para evitar conflito de callbacks. |
| `OnDayClick` | evento | - | Dispara ao selecionar um dia (`yyyy-mm-dd`). |
| `OnMonthChange` | evento | - | Dispara após mudança de mês (`yyyy-mm`). |

### 7.2) Propriedades gerais — `TD2BridgeCalendarioAno`

| Propriedade | Tipo | Padrão | Descrição |
| :--- | :--- | :--- | :--- |
| `DataSet` | `TDataSet` | `nil` | Fonte de dados com os eventos. |
| `FieldMap` | `TCalendarFieldMap` | - | Mapeamento dos campos do `DataSet`. |
| `Appearance` | `TCalendarAppearance` | - | Configuração visual. |
| `CurrentYear` | `Integer` | ano atual | Ajusta o ano e, em casos padrão, reposiciona `StartMonth` para Jan/1. |
| `StartMonth` | `TDateTime` | Jan/1 do ano | Mês inicial do intervalo exibido. |
| `MonthsToShow` | `Integer` | `12` | Quantidade de meses exibidos; também define o “pulo” do `CalendarYearNav`. |
| `CallbackPrefix` | `string` | `''` | Prefixo para evitar conflito de callbacks. |
| `OnMonthClick` | evento | - | Dispara ao clicar no nome do mês (`yyyy-mm`). |
| `OnDayClick` | evento | - | Dispara ao clicar em um dia (`yyyy-mm-dd`). |

### 7.3) `FieldMap` — `TCalendarFieldMap`

| Propriedade | Padrão | Descrição |
| :--- | :--- | :--- |
| `FieldID` | `ID` | Campo de identificação do evento (reservado; não usado no HTML atual). |
| `FieldDate` | `DATA` | Campo de data do evento (`TDateTime`). O componente usa apenas a data. |
| `FieldTitle` | `TITULO` | Texto do evento mostrado no dia. |
| `FieldColor` | `COR_EVENTO` | Cor do evento. Recomendado `#RRGGBB`. |

### 7.4) `Appearance` — `TCalendarAppearance`

| Propriedade | Tipo | Padrão | Descrição |
| :--- | :--- | :--- | :--- |
| `Theme` | enum | `ctDefault` | Tema geral (`ctDefault`, `ctDark`, `ctSoft`, `ctModern`). |
| `Font` | enum | `cfSansSerif` | Fonte (`cfSansSerif`, `cfSerif`, `cfMonospace`, `cfRoboto`, `cfOpenSans`, `cfLato`). |
| `HeaderColor` | string | `#ffffff` | Cor do cabeçalho. |
| `HeaderTextColor` | string | `#0d6efd` | Cor do texto/ícones do cabeçalho. |
| `FontSize` | Integer | `14` | Tamanho base da fonte (px). |
| `ShowHeader` | Boolean | `True` | Liga/desliga o cabeçalho. |
| `ShowNavigation` | Boolean | `True` | Mostra botões anterior/próximo/hoje. |
| `ShowTodayButton` | Boolean | `True` | Mostra o botão “Hoje”. |
| `ShowMonthText` | Boolean | `True` | Mostra o texto do mês/intervalo no header. |
| `ShowMonthPicker` | Boolean | `True` | (Mensal) mostra `input type=month` para escolher mês. |
| `ShowWeekDays` | Boolean | `True` | Mostra a linha de dias da semana. |
| `StartWeekOnMonday` | Boolean | `True` | `True` começa em Seg; `False` começa em Dom. |
| `ShowOutsideMonthDays` | Boolean | `True` | Mostra dias do mês anterior/próximo no grid. |
| `EventDisplayMode` | enum | `cedCountOnly` | Modo de exibição de eventos (veja 8). |
| `MaxPreviewItems` | Integer | `3` | Limite de itens por dia nos modos `cedPreviewList`/`cedBadges`. |

---

## 8) Modos de exibição de eventos

Configure em `Appearance.EventDisplayMode`:

- `cedCountOnly`: mostra apenas a bolinha com a quantidade de eventos no dia
- `cedPreviewList`: mostra linhas com títulos (texto simples), limitado por `MaxPreviewItems`
- `cedBadges`: mostra os títulos como badges (coloridos por `FieldColor` quando for `#...`), limitado por `MaxPreviewItems`

Se houver mais eventos do que `MaxPreviewItems`, o componente mostra um badge `+N` no dia.

---

## 10) Referências de código (API)

- Definições de classes e propriedades: [untCalendarioComponent.pas](./untCalendarioComponent.pas)
- Registro na paleta: [untCalendarioComponentRegister.pas](./untCalendarioComponentRegister.pas)
