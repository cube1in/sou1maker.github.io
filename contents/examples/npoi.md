# [NPOI](https://github.com/dotnetcore/NPOI)

## ExcelGenerator

```csharp
public static class ExcelGenerator
{
    internal static ISheet GenerateWithMergedRegion(IWorkbook workbook, string tableJson, bool enableStyle = true)
    {
        var sheet1 = workbook.CreateSheet("ExportData");

        var excelCells = Prepare(tableJson);

        #region [创建所需要的行]

        var need = excelCells.Max(cell => cell.LastRow);

        var rowsDictionary = new Dictionary<int, IRow>();
        for (var i = 0; i <= need; i++)
        {
            var row = sheet1.CreateRow(i);
            rowsDictionary.Add(i, row);
        }

        #endregion

        foreach (var excelCell in excelCells)
        {
            // 合并单元格
            sheet1.AddMergedRegion(new CellRangeAddress(excelCell.FirstRow, excelCell.LastRow, excelCell.FirstCol,
                excelCell.LastCol));

            // 取出 Row，并生成 Cell
            var cell = rowsDictionary[excelCell.FirstRow].CreateCell(excelCell.FirstCol);

            // 设置样式
            if (enableStyle && excelCell.Style != null)
            {
                SetStyle(workbook, cell, excelCell.Style);
                // if (cell.IsMergedCell)
                // {
                //     for (var i = excelCell.FirstRow; i <= excelCell.LastRow; i++)
                //     {
                //         for (var j = excelCell.FirstCol; j <= excelCell.LastCol; j++)
                //         {
                //             var mergedCell = rowsDictionary[i].CreateCell(j);
                //             SetStyle(workbook, mergedCell, excelCell.Style);
                //         }
                //     }
                // }
                // else
                // {
                //     SetStyle(workbook, cell, excelCell.Style);
                // }
            }

            switch (excelCell.Value)
            {
                case DateTime dateTime:
                    cell.SetCellValue(dateTime);
                    break;
                case IRichTextString richTextString:
                    cell.SetCellValue(richTextString);
                    break;
                case bool boolValue:
                    cell.SetCellValue(boolValue);
                    break;
                case double doubleValue:
                    cell.SetCellValue(doubleValue);
                    break;
                default:
                    cell.SetCellValue(excelCell.Value.ToString());
                    break;
            }
        }

        AutoSizeColumn(need, sheet1);
        return sheet1;
    }

    /// <summary>
    /// 设置样式
    /// </summary>
    /// <param name="workbook"></param>
    /// <param name="cell"></param>
    /// <param name="style"></param>
    private static void SetStyle(IWorkbook workbook, ICell cell, Style style)
    {
        var cellStyle = (XSSFCellStyle) workbook.CreateCellStyle();
        cellStyle.Alignment = HorizontalAlignment.Center;
        cellStyle.VerticalAlignment = VerticalAlignment.Center;

        // 自定义颜色值
        cellStyle.FillPattern = (FillPattern) style.FillPattern;
        cellStyle.SetFillForegroundColor(new XSSFColor(style.BackgroundColor.ToBytes()));
        cellStyle.SetFillBackgroundColor(new XSSFColor(style.ForegroundColor.ToBytes()));

        // if (style.BorderColor != null)
        // {
        //     cellStyle.BorderTop = BorderStyle.Thin;
        //     cellStyle.BorderBottom = BorderStyle.Thin;
        //     cellStyle.BorderLeft = BorderStyle.Thin;
        //     cellStyle.BorderRight = BorderStyle.Thin;
        //     
        //     cellStyle.SetBottomBorderColor(new XSSFColor(style.BorderColor.ToBytes()));
        //     cellStyle.SetBorderColor(BorderSide.TOP, new XSSFColor(style.BorderColor.ToBytes()));
        //     cellStyle.SetBorderColor(BorderSide.BOTTOM, new XSSFColor(style.BorderColor.ToBytes()));
        //     cellStyle.SetBorderColor(BorderSide.LEFT, new XSSFColor(style.BorderColor.ToBytes()));
        //     cellStyle.SetBorderColor(BorderSide.RIGHT, new XSSFColor(style.BorderColor.ToBytes()));
        // }

        #region [字体]

        var font = (XSSFFont) workbook.CreateFont();

        // 字体颜色
        var fontColor = new XSSFColor();
        fontColor.SetRgb(style.Color.ToBytes());
        font.SetColor(fontColor);

        // 字体重量
        font.Boldweight = style.FontBoldWeight;

        // 字体大小
        font.FontHeightInPoints = short.Parse(style.FontSize.TrimEnd('p', 'x'));

        cellStyle.SetFont(font);

        cell.CellStyle = cellStyle;

        #endregion
    }

    /// <summary>
    /// 自适应列宽
    /// </summary>
    /// <param name="max"></param>
    /// <param name="sheet"></param>
    private static void AutoSizeColumn(int max, ISheet sheet)
    {
        for (var columnNum = 0; columnNum < max; columnNum++)
        {
            var columnWidth = sheet.GetColumnWidth(columnNum) / 256;
            for (var rowNum = 1; rowNum <= sheet.LastRowNum; rowNum++)
            {
                var currentRow = sheet.GetRow(rowNum) ?? sheet.CreateRow(rowNum);

                if (currentRow.GetCell(columnNum) == null) continue;

                var currentCell = currentRow.GetCell(columnNum);
                var length = Encoding.Default.GetBytes(currentCell.ToString()!).Length;
                if (columnWidth < length)
                {
                    columnWidth = length;
                }
            }

            sheet.SetColumnWidth(columnNum, columnWidth * 256);
        }
    }

    /// <summary>
    /// 前端转换, RowSpan => Position
    /// </summary>
    /// <param name="tableJson"></param>
    /// <returns></returns>
    private static ICollection<ExcelCell> Prepare(string tableJson)
    {
        var excel = JsonConvert.DeserializeObject<List<List<ExcelCell>>>(tableJson);

        var excelCells = new Collection<ExcelCell>();
        var columnIndexDict = new Dictionary<int, int>();
        for (int i = 0, rowIndex = 0; i < excel.Count; i++)
        {
            var row = excel[i];
            if (!columnIndexDict.ContainsKey(i)) columnIndexDict[i] = 0;
            for (int j = 0, colIndex = 0; j < row.Count; j++)
            {
                var cell = row[j];
                // 这里需要记录开始列
                if (cell.RowSpan > 1)
                {
                    // 存在span 则需要设置下一行的开始列
                    for (var k = i + 1 ; k < i + cell.RowSpan; k++)
                    {
                        if (!columnIndexDict.ContainsKey(k)) columnIndexDict[k] = 0;
                        columnIndexDict[k] += 1;
                    }
                }

                cell.FirstRow = i;
                cell.FirstCol = columnIndexDict[i] + colIndex; 
                cell.LastRow = cell.FirstRow + cell.RowSpan - 1;
                cell.LastCol = cell.FirstCol + cell.ColSpan - 1;

                excelCells.Add(cell);

                colIndex += cell.ColSpan;
            }

            if (rowIndex == i)
            {
                rowIndex += row.Max(o => o.RowSpan);
            }

        }

        return excelCells;
    }
}


[SuppressMessage("ReSharper", "UnusedAutoPropertyAccessor.Global")]
[SuppressMessage("ReSharper", "ClassNeverInstantiated.Global")]
public class ExcelCell
{

    #region [前端结构]

    /// <summary>
    /// 跨行
    /// </summary>
    public int RowSpan { get; set; }

    /// <summary>
    /// 跨列
    /// </summary>
    public int ColSpan { get; set; }

    /// <summary>
    /// 是否是表头(默认TD)
    /// <example>TH(表头)</example>
    /// <example>TD(单元格)</example>
    /// </summary>
    public string Type { get; set; } = "TD";

    #endregion

    /// <summary>
    /// 起始行号
    /// </summary>
    public int FirstRow { get; set; }

    /// <summary>
    /// 终止行号
    /// </summary>
    public int LastRow { get; set; }

    /// <summary>
    /// 起始列号
    /// </summary>
    public int FirstCol { get; set; }

    /// <summary>
    /// 终止列号
    /// </summary>
    public int LastCol { get; set; }

    /// <summary>
    /// 样式
    /// </summary>
    public Style? Style { get; set; }

    /// <summary>
    /// 值
    /// </summary>
    public object Value { get; set; } = null!;
}

[SuppressMessage("ReSharper", "MemberCanBePrivate.Global")]
[SuppressMessage("ReSharper", "UnusedAutoPropertyAccessor.Global")]
public class Style
{
    /// <summary>
    /// 背景填充色
    /// </summary>
    public Color ForegroundColor { get; set; } = "rgb(255, 255, 255)";

    /// <summary>
    /// 填充图案的颜色 
    /// </summary>
    public Color BackgroundColor { get; set; } = "rgb(255, 255, 255)";

    /// <summary>
    /// 边框颜色
    /// </summary>
    public Color? BorderColor { get; set; }

    /// <summary>
    /// 字体颜色
    /// </summary>
    public Color Color { get; set; } = "rgb(0, 0, 0)";

    /// <summary>
    /// 字重
    /// </summary>
    public short FontBoldWeight { get; set; } = (short) NPOI.SS.UserModel.FontBoldWeight.Normal;

    /// <summary>
    /// 字号大小(FontHeightInPoints)
    /// </summary>
    public string FontSize { get; set; } = "12px";

    /// <summary>
    /// 填充样式(默认0)
    /// </summary>
    public short FillPattern { get; set; } = 0;
}

[SuppressMessage("ReSharper", "MemberCanBePrivate.Global")]
public class Color
{
    private static readonly Regex RgbRegex = new(@"^rgb\((?<red>[0-9]{1,3}),\s*(?<green>[0-9]{1,3}),\s*(?<blue>[0-9]{1,3})\)$", RegexOptions.Compiled | RegexOptions.IgnoreCase);

    /// <summary>
    /// 红
    /// </summary>
    public byte Red { get; set; }

    /// <summary>
    /// 绿
    /// </summary>
    public byte Green { get; set; }

    /// <summary>
    /// 蓝
    /// </summary>
    public byte Blue { get; set; }

    public byte[] ToBytes() => new[] {Red, Green, Blue};


    public static implicit operator string (Color input)
    {
        return $"rgb({input.Red}, {input.Green}, {input.Blue})";
    }

    public static implicit operator Color (string input)
    {
        var color = new Color();
        var match = RgbRegex.Match(input);

        if (!match.Success)
        {
            throw new ArgumentException($"Error converting value \"{input}\" to type '{typeof(Color)}'");
        }

        color.Red = byte.Parse(match.Groups["red"].Value);
        color.Green = byte.Parse(match.Groups["green"].Value);
        color.Blue = byte.Parse(match.Groups["blue"].Value);

        return color;
    }
}
```
