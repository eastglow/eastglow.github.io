---
layout: post
title:  "[Spring]Apache POI를 이용한 Excel Download"
date:   2019-03-07 19:00:00
author: EastGlow
categories: Back-end
---
## 서론
이전에는 특정 페이지를 엑셀로 다운로드 받으려면 JSP를 이용하곤 했다.

```
<%
    String fileName = "Excel File_"+DateTime.getDateTimeMinSec()+".xls";
    response.setHeader("Content-Disposition", "attachment; filename="+new String(fileName.getBytes("utf-8"),"8859_1"));
    response.setHeader("Content-Description","JSP Generated Data");
%>
```

위와 같이 JSP 상단에 적어주면 html 코드들이 엑셀로 저장이 된다. 간편하긴 하지만 \<table\>을 이용하여 표를 다 그려줘야하고 제일 치명적인 단점이 엑셀 97~2003버전(xls)으로만 저장이 된다는 것이다. 그래서 2003버전 이상에서 파일을 열면 경고창이 하나 뜨고(그냥 무시하고 진행하면 열리긴 한다.) 수정에 제약이 있다. 매크로라든지 2003버전 이상에서만 작동하는 것들을 쓰지 못하는 것으로 알고 있다.

그래서 이번에 전자정부 프레임워크(Spring 4.x 기반)를 이용한 프로젝트를 진행할 땐 아예 공통으로 쓸 수 있는 Excel Download 클래스를 하나 만들기로 했다.

찾아보니 이미 전자정부 프레임워크 내부에도 관련된 서비스가 있어서 참고하여 만들었다.

## 본론
### context-common.xml
```
<bean id="excelView" class="egovframework.test.egov.util.ExcelView" />
```
어차피 Excel Download에 필요한 클래스는 매번 new()를 통해 생성할 필요도 없고 하나만 있으면 되고 bean id로 등록해두면 사용하기 편리하여 context-common.xml이라는 공통 bean들이 모여있는 곳에 선언해두었다. Spring이 올라가면서 `exccelView`라는 Excel Download 클래스가 하나 만들어지게 된다.

### ExcelView.java
```
public class ExcelView extends AbstractView {
    private static final Logger LOGGER  = LoggerFactory.getLogger(ExcelView.class);

    /** The content type for an Excel response */
     private static final String CONTENT_TYPE_XLSX = "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet";

    /**
     * Default Constructor. Sets the content type of the view for excel files.
     */
    public ExcelView() {
    }

    @Override
    protected boolean generatesDownloadContent() {
        return true;
    }

    /**
     * Renders the Excel view, given the specified model.
     */
    @Override
    protected final void renderMergedOutputModel(Map<String, Object> model, HttpServletRequest request, HttpServletResponse response) throws Exception {

        XSSFWorkbook workbook = new XSSFWorkbook();
        LOGGER.debug("Created Excel Workbook from scratch");

        setContentType(CONTENT_TYPE_XLSX);

        buildExcelDocument(model, workbook, request, response);
        
        // Set the filename
        String sFilename = "";
        if(model.get("filename") != null){
            sFilename = (String)model.get("filename");
        }else if(request.getAttribute("filename") != null){
            sFilename = (String)request.getAttribute("filename");
        }else{
            sFilename = getClass().getSimpleName();
         }

        response.setContentType(getContentType());
        
        String header = request.getHeader("User-Agent");
        sFilename = sFilename.replaceAll("\r","").replaceAll("\n","");
        if(header.contains("MSIE") || header.contains("Trident") || header.contains("Chrome")){
            sFilename = URLEncoder.encode(sFilename,"UTF-8").replaceAll("\\+","%20");
            response.setHeader("Content-Disposition","attachment;filename="+sFilename+".xlsx;");
        }else{
            sFilename = new String(sFilename.getBytes("UTF-8"),"ISO-8859-1");
            response.setHeader("Content-Disposition","attachment;filename=\""+sFilename + ".xlsx\"");
        }
        
        // Flush byte array to servlet output stream.
        ServletOutputStream out = response.getOutputStream();
        out.flush();
        workbook.write(out);
        out.flush();

        // Don't close the stream - we didn't open it, so let the container handle it.
        // http://stackoverflow.com/questions/1829784/should-i-close-the-servlet-outputstream
    }
    
    @SuppressWarnings("unchecked")
    protected void buildExcelDocument(Map model, XSSFWorkbook wb, HttpServletRequest req, HttpServletResponse resp) throws Exception {
        Map<String, Object> dataMap = (Map<String, Object>) model.get("dataMap");
        XSSFCell cell = null;
 
        String sheetNm = (String) dataMap.get("sheetNm"); // 엑셀 시트 이름
        
        String[] columnArr = (String[]) dataMap.get("columnArr"); // 각 컬럼 이름
        String[] columnVarArr = (String[]) dataMap.get("columnVarArr"); // 각 컬럼의 변수 이름
        
        List<EgovMap> dataList = (List<EgovMap>) dataMap.get("list"); // 데이터가 담긴 리스트 
        
        CellStyle cellStyle = wb.createCellStyle(); // 제목셀의 셀스타일
        cellStyle.setWrapText(true); // 줄 바꿈            
        cellStyle.setFillForegroundColor(HSSFColor.GREY_25_PERCENT.index); // 셀 색상
        cellStyle.setFillPattern(HSSFCellStyle.SOLID_FOREGROUND); // 셀 색상 패턴
        cellStyle.setAlignment(HSSFCellStyle.ALIGN_CENTER); // 셀 가로 정렬
        cellStyle.setVerticalAlignment(HSSFCellStyle.VERTICAL_CENTER); // 셀 세로 정렬
        cellStyle.setDataFormat((short)0x31); // 셀 데이터 형식
        cellStyle.setBorderRight(HSSFCellStyle.BORDER_DOUBLE);
        cellStyle.setBorderLeft(HSSFCellStyle.BORDER_DOUBLE);
        cellStyle.setBorderTop(HSSFCellStyle.BORDER_DOUBLE);
        cellStyle.setBorderBottom(HSSFCellStyle.BORDER_DOUBLE);
        
        // 셀 폰트색상, bold처리
        Font font = wb.createFont();
        font.setColor(HSSFColor.WHITE.index);
        font.setBoldweight(HSSFFont.BOLDWEIGHT_BOLD);
        cellStyle.setFont(font);
        
        CellStyle cellStyle2 = wb.createCellStyle(); // 데이터셀의 셀스타일
        cellStyle2.setWrapText(true); // 줄 바꿈           
        cellStyle2.setAlignment(HSSFCellStyle.ALIGN_CENTER); // 셀 가로 정렬
        cellStyle2.setVerticalAlignment(HSSFCellStyle.VERTICAL_CENTER); // 셀 세로 정렬
        cellStyle2.setDataFormat((short)0x31); // 셀 데이터 형식
        cellStyle2.setBorderRight(HSSFCellStyle.BORDER_THIN);
        cellStyle2.setBorderLeft(HSSFCellStyle.BORDER_THIN);
        cellStyle2.setBorderTop(HSSFCellStyle.BORDER_THIN);
        cellStyle2.setBorderBottom(HSSFCellStyle.BORDER_THIN);
        
        XSSFSheet sheet = wb.createSheet(sheetNm);
        sheet.setDefaultColumnWidth(12);
        
        // 컬럼명 삽입
        for(int i=0; i<columnArr.length; i++){
            setText(getCell(sheet, 0, i), columnArr[i]);
            getCell(sheet, 0, i).setCellStyle(cellStyle);
            sheet.autoSizeColumn(i);
            int columnWidth = (sheet.getColumnWidth(i))*5;
            sheet.setColumnWidth(i, columnWidth);
            
            if(dataList.size() < 1){
                cell = getCell(sheet, 1, i);
                if(i==0){
                    setText(cell, "등록된 정보가 없습니다.");
                }
                cell.setCellStyle(cellStyle2);
            }
        }
        
        if(dataList.size() > 0){ // 저장된 데이터가 있을때
            // 리스트 데이터 삽입
            for (int i = 0; i<dataList.size(); i++) {
                EgovMap dataEgovMap = dataList.get(i);
                
                // 맨 앞 컬럼인 "번호"는 idx라는 이름으로 여기서 생성하여 넣어준다.
                dataEgovMap.put("idx", (i+1)+""); 
                
                for(int j=0; j<columnVarArr.length; j++){
                    String data = (String) dataEgovMap.get(columnVarArr[j]);
                    cell = getCell(sheet, 1 + i, j);
                    setText(cell, data);
                    cell.setCellStyle(cellStyle2);
                }
            }
        }else{ // 저장된 데이터가 없으면 셀 병합
            // 셀 병합(시작열, 종료열, 시작행, 종료행)
            sheet.addMergedRegion(new CellRangeAddress(1, 1, 0, columnArr.length-1));
        }
    }

    /**
     * Convenient method to obtain the cell in the given sheet, row and column.
     * 
     * <p>Creates the row and the cell if they still doesn't already exist.
     * Thus, the column can be passed as an int, the method making the needed downcasts.</p>
     * 
     * @param sheet a sheet object. The first sheet is usually obtained by workbook.getSheetAt(0)
     * @param row thr row number
     * @param col the column number
     * @return the XSSFCell
     */
    protected XSSFCell getCell(XSSFSheet sheet, int row, int col) {
        XSSFRow sheetRow = sheet.getRow(row);
        if (sheetRow == null) {
            sheetRow = sheet.createRow(row);
        }
        XSSFCell cell = sheetRow.getCell((short) col);
        if (cell == null) {
            cell = sheetRow.createCell((short) col);
        }
        return cell;
    }

    /**
     * Convenient method to set a String as text content in a cell.
     * 
     * @param cell the cell in which the text must be put
     * @param text the text to put in the cell
     */
    protected void setText(XSSFCell cell, String text) {
        cell.setCellType(XSSFCell.CELL_TYPE_STRING);
        cell.setCellValue(text);
    }            
}
```

사실 전자정부 프레임워크에서 제공하는 `AbstractPOIExcelView`라는 추상 클래스가 있는데 이 친구가 문제가 하나 있었다. 위 소스에서도 볼 수 있는 `renderMergedOutputModel`라는 메서드가 하나 있다. 이 메서드의 역할은 완성된 엑셀 데이터를 파일로 내보내주는 것인데 한글 파일명으로 파일을 받으면 깨지는 것이다. 물론, 영어로 파일명을 지으면 잘 저장된다.

뭐가 문제인지 찾아보니 `AbstractPOIExcelView`안에 있는 `renderMergedOutputModel` 코드가 문제였다.

#### protected final void renderMergedOutputModel(Map<String, Object> model, HttpServletRequest request, HttpServletResponse response)
```
.
.
.
// Set the filename
String sFilename = "";
if(model.get("filename") != null){
    sFilename = (String)model.get("filename");
}else if(request.getAttribute("filename") != null){
    sFilename = (String)request.getAttribute("filename");
}else{
    sFilename = getClass().getSimpleName();
}

// Set the content type.
response.setContentType(getContentType());
response.setHeader("Content-Disposition", "attachment; filename=\"" + sFilename + ".xlsx\"");
.
.
.
```

`renderMergedOutputModel`는 `AbstractView` 라는 곳에 있는 추상 메서드를 상속 받아서 구현한 상태였는데 `// Set the content type` 주석 부분부터 보면 response에 컨텐츠 타입과 헤더를 담는 것을 볼 수 있다.

그런데 한글이 파일명으로 들어왔을 때에 대한 처리가 전혀 되어 있지 않았다. 그러다보니 한글 파일명을 넘기면 깨지거나 아예 올바른 엑셀 파일로 저장이 되지 않았다.

그럼 이 메서드를 `context-common.xml`에서 선언한 `ExcelView`에서 오버라이딩하여 다시 구현하면 되지 않을까...라고 생각하여 봤는데 `AbstractPOIExcelView`에서 이미 final 메서드로 선언이 되어 있어서 상속 받아 오버라이딩 할 수 없는 상태였다... (내가 아직 모자라서 방법을 모르는 걸수도 있지만 내가 아는 한 final로 선언된 친구들은 상속 받아 재구현할 수 없다.)

그래서 그냥 `AbstractPOIExcelView`를 `ExcelView`에서 상속 받지 않고 `AbstractPOIExcelView`가 상속 받고 있던 `AbstractView`를 직접 상속 받고 `AbstractPOIExcelView`안에 있던 코드를 모두 `ExcelView`로 옮겨적었다. 정리하자면,

원래는 `AbstractView` -> `AbstractPOIExcelView` -> `ExcelView` 였는데
바꿔서 `AbstractView` -> `ExcelView` 로 바뀐 것이다.

그렇게 나온 코드가 위에 언급한 `ExcelView.java` 코드이다. `renderMergedOutputModel`를 실행하게 되면 `buildExcelDocument`가 실행되게 된다. `buildExcelDocument`가 직접적으로 엑셀 파일을 만들어주는 메서드이다.

`buildExcelDocument`에서 컨트롤러로부터 넘겨받은 데이터들을 가지고 엑셀 파일을 만들어주고 있다. 크게 어려운 코드는 없어서 설명은 하지 않겠다.

여차저차해서 엑셀 파일을 만들고 `renderMergedOutputModel` 끝 부분에서 파일명을 가지고 엑셀 파일을 만들어서 내보내준다. 참고로 컨트롤러에서 넘어온 맵 안에 `filename`이라는 변수가 없으면 `ExcelView`라는 클래스 이름으로 파일이 저장 된다.

### Controller.java
```
@RequestMapping(value={"/boardlistexcel.do"})
public ModelAndView boardListExcel(BoardVO paramVO, HttpServletRequest request, HttpServletResponse response, HttpSession session) throws Exception{
    ModelAndView mav = new ModelAndView("excelView");
    Map<String, Object> dataMap = new HashMap<String, Object>();

    List<?> list = boardSvc.getBoardAllList(paramVO);
    String filename = "게시물 목록_"+DateTime.getDateTimeMinSec();
    String[] columnArr = {"번호", "제목", "작성자", "작성일"};
    String[] columnVarArr = {"idx", "title", "author", "regDate"};
            
    dataMap.put("columnArr", columnArr);
    dataMap.put("columnVarArr", columnVarArr);
    dataMap.put("sheetNm", "게시물 목록");    
    dataMap.put("list", list);
    
    mav.addObject("dataMap", dataMap);
    mav.addObject("filename", filename);
    
    return mav;
}
```

컨트롤러에서 어떤 데이터를 넘겨주든 그건 작성자의 마음이기 때문에 꼭 위 코드처럼 넘겨줄 필요는 없다.

나는 `list`(EgovMap 형태의 데이터가 담긴 리스트), `filename`(파일명), `columnArr`(컬럼명), `columnVarArr`(DB에서 가져온 list 안에 있는 EgovMap에서 get할 때 필요한 변수명), `sheetNm`(시트명) 총 5개의 파라미터를 전달해준다.

이렇게 하면 Apache POI를 이용한 엑셀(xlsx)파일 저장하기가 완료된다.

>참고 자료: http://www.egovframe.go.kr/wiki/doku.php?id=egovframework:rte3:fdl:excel
