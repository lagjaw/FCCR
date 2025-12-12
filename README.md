private PdfPTable buildAbcTable(Notation notation) {
    Font headerFont = new Font(Font.HELVETICA, 9, Font.BOLD, Color.WHITE);
    PdfPTable abcTable = new PdfPTable(3); // Code, Valeur, Risk
    abcTable.setWidthPercentage(100);
    abcTable.setSpacingAfter(12);
    abcTable.setWidths(new float[]{1.2f, 1.8f, 1.0f});
    abcTable.setKeepTogether(true);

    // En-têtes
    Stream.of("CODE", "VALEUR", "NIVEAU DE RISQUE").forEach(col -> {
        PdfPCell cell = new PdfPCell(new Phrase(col, headerFont));
        cell.setBackgroundColor(HEADER_BG);
        cell.setHorizontalAlignment(Element.ALIGN_CENTER);
        cell.setVerticalAlignment(Element.ALIGN_MIDDLE);
        cell.setPadding(6);
        cell.setBorderWidth(0.5f);
        cell.setBorderColor(Color.DARK_GRAY);
        abcTable.addCell(cell);
    });

    boolean alternate = false;

    for (NotationLine line : notation.getNotationLines()) {
        Color bgColor = alternate ? ZEBRA_COLOR : Color.WHITE;
        alternate = !alternate;

        abcTable.addCell(createPdfCell(line.getCode(), bgColor, 9));
        abcTable.addCell(createPdfCell(
                line.getValue() != null ? line.getValue() : "-", bgColor, 9));
        abcTable.addCell(createPdfCell(
                line.getRisk() != null ? line.getRisk() : "-", bgColor, 9));
    }

    return abcTable;
}
------------

// --- Section Anti-Bribery & Corruption ---
if ("YES".equalsIgnoreCase(String.valueOf(notation.getFlagABC()))) {
    Paragraph abcTitle = new Paragraph("Anti-Bribery & Corruption", SECTION_TITLE_FONT);
    abcTitle.setSpacingBefore(10);
    abcTitle.setSpacingAfter(8);
    document.add(abcTitle);

    document.add(buildAbcTable(notation));
}

