package com.sahambank.fccr.core.service.Impl;

import com.lowagie.text.*;
import com.lowagie.text.pdf.*;
import com.sahambank.fccr.core.dto.ConfigMoteurDetail.NotationResponse;
import com.sahambank.fccr.core.dto.UserInfo.Action;
import com.sahambank.fccr.core.entities.*;
import com.sahambank.fccr.core.repository.FieldConfigurationRepository;
import com.sahambank.fccr.core.repository.NotationRepository;
import com.sahambank.fccr.core.repository.SegmentRepository;
import com.sahambank.fccr.core.service.PdfService;
import jakarta.persistence.EntityNotFoundException;
import org.springframework.stereotype.Service;

import java.awt.*;
import java.io.ByteArrayOutputStream;
import java.io.InputStream;
import java.time.format.DateTimeFormatter;
import java.util.Optional;
import java.util.stream.Collectors;
import java.util.stream.Stream;

@Service
public class PdfServiceImpl implements PdfService {

    private final NotationRepository notationRepository;
    private final SegmentRepository segmentRepository;
    private final FieldConfigurationRepository fieldConfigurationRepository;
    private final CurrentUserService currentUserService;
    private final AuthorizationService authorizationService;

    // ========== Constantes de style ==========
    private static final Color PRIMARY_COLOR = new Color(10, 35, 66);
    private static final Color ACCENT_COLOR = new Color(192, 0, 0); // Rouge foncé pour les titres
    private static final Color HEADER_BG = new Color(10, 35, 66);
    private static final Color ZEBRA_COLOR = new Color(240, 240, 240);
    private static final Font TITLE_FONT = new Font(Font.HELVETICA, 16, Font.BOLD, ACCENT_COLOR);
    private static final Font SECTION_TITLE_FONT = new Font(Font.HELVETICA, 11, Font.BOLD, ACCENT_COLOR);
    private static final Font LABEL_FONT = new Font(Font.HELVETICA, 10, Font.BOLD, PRIMARY_COLOR);
    private static final Font VALUE_FONT = new Font(Font.HELVETICA, 10, Font.NORMAL, Color.BLACK);
    private static final Font HEADER_FONT = new Font(Font.HELVETICA, 10, Font.BOLD, Color.WHITE);
    private static final DateTimeFormatter DATE_FORMATTER = DateTimeFormatter.ofPattern("dd/MM/yyyy HH:mm:ss");

    public PdfServiceImpl(NotationRepository notationRepository, SegmentRepository segmentRepository,
                         FieldConfigurationRepository fieldConfigurationRepository,
                         CurrentUserService currentUserService,
                         AuthorizationService authorizationService) {
        this.notationRepository = notationRepository;
        this.segmentRepository = segmentRepository;
        this.fieldConfigurationRepository = fieldConfigurationRepository;
        this.authorizationService = authorizationService;
        this.currentUserService = currentUserService;
    }

    @Override
    public byte[] generatePdfForNotation(Long notationId) {
        PortalUser currentUser = currentUserService.getCurrentPortalUser();
        authorizationService.authorize(currentUser.getEmail(), Action.EXPORT_PDF);

        Notation notation = notationRepository.findById(notationId)
                .orElseThrow(() -> new EntityNotFoundException("Notation introuvable avec l'ID : " + notationId));

        ByteArrayOutputStream out = new ByteArrayOutputStream();
        
        // === PAGE SIZE: Landscape A4 avec marges réduites ===
        Rectangle pageSize = PageSize.A4.rotate(); // Landscape
        Document document = new Document(pageSize, 30, 30, 50, 30); // left, right, top, bottom margins
        PdfWriter writer = PdfWriter.getInstance(document, out);

        // Logo et header en haut de page
        writer.setPageEvent(new PdfPageEventHelper() {
            @Override
            public void onEndPage(PdfWriter writer, Document document) {
                PdfContentByte cb = writer.getDirectContent();
                try {
                    InputStream is = getClass().getResourceAsStream("/Assets/logo-SAHAM.jfif");
                    if (is != null) {
                        byte[] logoBytes = is.readAllBytes();
                        Image logo = Image.getInstance(logoBytes);
                        logo.scaleToFit(80, 80);
                        logo.setAbsolutePosition(document.leftMargin(), document.top() + 10);
                        cb.addImage(logo);
                    }
                } catch (Exception e) {
                    // Log ou ignorer si le logo n'est pas trouvé
                }
                
                // Ligne de séparation en haut
                cb.setColorStroke(PRIMARY_COLOR);
                cb.setLineWidth(1f);
                cb.moveTo(document.leftMargin(), document.top() - 50);
                cb.lineTo(document.right() - document.rightMargin(), document.top() - 50);
                cb.stroke();
            }
        });

        document.open();

        // --- Badge "C2 - CONFIDENTIAL" ---
        Paragraph confidential = new Paragraph("C2 - CONFIDENTIAL", 
                new Font(Font.HELVETICA, 9, Font.BOLD, PRIMARY_COLOR));
        confidential.setSpacingAfter(8);
        document.add(confidential);

        // --- Titre principal ---
        Paragraph title = new Paragraph("Financial Crime Rating Report", TITLE_FONT);
        title.setAlignment(Element.ALIGN_CENTER);
        title.setSpacingAfter(15);
        document.add(title);

        // --- Info Client (format texte structuré, 2 colonnes) ---
        document.add(buildClientInfoSection(notation, currentUser));

        // --- Section Rating Result (texte structuré) ---
        document.add(buildRatingResultSection(notation, currentUser));

        // --- Section Rating Warnings (titre avec ligne) ---
        Paragraph warningsTitle = new Paragraph("Rating Warnings", SECTION_TITLE_FONT);
        warningsTitle.setSpacingBefore(10);
        warningsTitle.setSpacingAfter(8);
        document.add(warningsTitle);

        // --- Tableau des lignes de notation ---
        document.add(buildNotationTable(notation));

        // --- Section Applicable Business Rules ---
        Paragraph rulesTitle = new Paragraph("Applicable business rules", SECTION_TITLE_FONT);
        rulesTitle.setSpacingBefore(10);
        rulesTitle.setSpacingAfter(8);
        document.add(rulesTitle);

        // --- Tableau des règles ---
        document.add(buildProductTable(notation));

        // --- Pied de page ---
        Paragraph footer = new Paragraph("C2 - CONFIDENTIAL", 
                new Font(Font.HELVETICA, 8, Font.NORMAL, PRIMARY_COLOR));
        footer.setAlignment(Element.ALIGN_RIGHT);
        footer.setSpacingBefore(10);
        document.add(footer);

        document.close();
        return out.toByteArray();
    }

    // ========== Construction des sections ==========

    /**
     * Construit la section info client en format 2 colonnes (texte structuré, pas tableau)
     */
    private PdfPTable buildClientInfoSection(Notation notation, PortalUser currentUser) {
        PdfPTable mainTable = new PdfPTable(2);
        mainTable.setWidthPercentage(100);
        mainTable.setSpacingAfter(12);
        mainTable.setWidths(new float[]{1, 1});
        mainTable.getDefaultCell().setBorder(Rectangle.NO_BORDER);
        mainTable.getDefaultCell().setPadding(3);

        // --- Colonne gauche ---
        PdfPCell leftCell = new PdfPCell();
        leftCell.setBorder(Rectangle.NO_BORDER);
        leftCell.setPadding(0);

        Paragraph leftContent = new Paragraph();
        leftContent.add(addInfoLine("Third party", ""));

        // mapping depuis l'ancienne buildInfoTable
        leftContent.add(addInfoLine("Client", notation.getClientCode()));
        leftContent.add(addInfoLine("Segment", notation.getSegmentCode()));
        leftContent.add(addInfoLine("Date de création", 
                notation.getCreatedAt() != null ? notation.getCreatedAt().format(DATE_FORMATTER) : "-"));

        String userName = (currentUser.getFirstName() != null ? currentUser.getFirstName() : "") + " " +
                (currentUser.getLastName() != null ? currentUser.getLastName() : "");
        leftContent.add(addInfoLine("User name", userName.trim()));

        String roles = currentUser.getRoles().stream()
                .map(role -> role.getCode().name())
                .collect(Collectors.joining(", "));
        leftContent.add(addInfoLine("Roles", roles));

        leftContent.add(addInfoLine("Organization", getCurrentUserEntiteCode()));

        leftCell.addElement(leftContent);
        mainTable.addCell(leftCell);

        // --- Colonne droite ---
        PdfPCell rightCell = new PdfPCell();
        rightCell.setBorder(Rectangle.NO_BORDER);
        rightCell.setPadding(0);

        Paragraph rightContent = new Paragraph();
        rightContent.add(addInfoLine("Date & time", 
                notation.getCreatedAt() != null ? notation.getCreatedAt().format(DATE_FORMATTER) : "-"));
        rightContent.add(addInfoLine("Rating (note)", 
                notation.getNote() != null ? notation.getNote().toString() : "-"));
        rightContent.add(addInfoLine("ABC Flag", 
                notation.getFlagABC() != null ? notation.getFlagABC().toString() : "-"));

        rightCell.addElement(rightContent);
        mainTable.addCell(rightCell);

        return mainTable;
    }

    /**
     * Construit la section Rating Result (texte structuré, 2 colonnes)
     */
    private PdfPTable buildRatingResultSection(Notation notation, PortalUser currentUser) {
        PdfPTable mainTable = new PdfPTable(2);
        mainTable.setWidthPercentage(100);
        mainTable.setSpacingAfter(12);
        mainTable.setWidths(new float[]{1, 1});
        mainTable.getDefaultCell().setBorder(Rectangle.NO_BORDER);
        mainTable.getDefaultCell().setPadding(3);

        PdfPCell titleCell = new PdfPCell(new Phrase("Rating Result", SECTION_TITLE_FONT));
        titleCell.setColspan(2);
        titleCell.setBorder(Rectangle.NO_BORDER);
        titleCell.setPaddingBottom(5);
        mainTable.addCell(titleCell);

        // --- Colonne gauche ---
        PdfPCell leftCell = new PdfPCell();
        leftCell.setBorder(Rectangle.NO_BORDER);
        leftCell.setPadding(0);

        Paragraph leftContent = new Paragraph();
        leftContent.add(addInfoLine("Third Party ID rated", notation.getClientCode()));
        leftContent.add(addInfoLine("Date, User name",
                (notation.getCreatedAt() != null ? notation.getCreatedAt().format(DATE_FORMATTER) : "-")
                        + " - " +
                ((currentUser.getFirstName() != null ? currentUser.getFirstName() : "") + " " +
                 (currentUser.getLastName() != null ? currentUser.getLastName() : "")).trim()));
        leftContent.add(addInfoLine("Role, Organization",
                currentUser.getRoles().stream().map(r -> r.getCode().name()).collect(Collectors.joining(", "))
                        + " - " + getCurrentUserEntiteCode()));
        leftContent.add(addInfoLine("Methodology", "V3 - Target"));

        leftCell.addElement(leftContent);
        mainTable.addCell(leftCell);

        // --- Colonne droite ---
        PdfPCell rightCell = new PdfPCell();
        rightCell.setBorder(Rectangle.NO_BORDER);
        rightCell.setPadding(0);

        Paragraph rightContent = new Paragraph();
        String note = notation.getNote() != null ? notation.getNote().toString() : "-";
        rightContent.add(addInfoLine("Computed Rating", note));
        rightContent.add(addInfoLine("Model's Rating", note));
        rightContent.add(addInfoLine("Rating Status", "Validated"));
        rightContent.add(addInfoLine("ABC Flag",
                notation.getFlagABC() != null ? notation.getFlagABC().toString() : "-"));

        rightCell.addElement(rightContent);
        mainTable.addCell(rightCell);

        return mainTable;
    }

    /**
     * Construit le tableau des lignes de notation (Rating Warnings)
     */
    private PdfPTable buildNotationTable(Notation notation) {
        Font headerFont = new Font(Font.HELVETICA, 9, Font.BOLD, Color.WHITE);
        PdfPTable table = new PdfPTable(7);
        table.setWidthPercentage(100);
        table.setSpacingAfter(12);
        table.setWidths(new float[]{1.2f, 1.5f, 1.2f, 1.2f, 0.8f, 0.8f, 0.8f});

        // En-têtes avec fond bleu foncé
        Stream.of("Code", "Libelle", "Valeur", "Risque", "Poids L", "Poids ML", "Poids MH").forEach(col -> {
            PdfPCell cell = new PdfPCell(new Phrase(col, headerFont));
            cell.setBackgroundColor(HEADER_BG);
            cell.setHorizontalAlignment(Element.ALIGN_CENTER);
            cell.setVerticalAlignment(Element.ALIGN_MIDDLE);
            cell.setPadding(6);
            table.addCell(cell);
        });

        // Données avec zébrage
        boolean alternate = false;
        for (NotationLine line : notation.getNotationLines()) {
            Optional<FieldConfiguration> fieldConfiguration = fieldConfigurationRepository.findByCode(line.getCode());
            String libelle = fieldConfiguration.map(FieldConfiguration::getLibelle).orElse("N/A");

            Color bgColor = alternate ? ZEBRA_COLOR : Color.WHITE;
            alternate = !alternate;

            table.addCell(createPdfCell(line.getCode(), bgColor, 9));
            table.addCell(createPdfCell(libelle, bgColor, 9));
            table.addCell(createPdfCell(line.getValue(), bgColor, 9));
            table.addCell(createPdfCell(line.getRisk(), bgColor, 9));
            table.addCell(createPdfCell(String.valueOf(line.getWeightL()), bgColor, 9));
            table.addCell(createPdfCell(String.valueOf(line.getWeightMl()), bgColor, 9));
            table.addCell(createPdfCell(String.valueOf(line.getWeightMh()), bgColor, 9));
        }

        return table;
    }

    /**
     * Construit le tableau des produits (Applicable business rules)
     */
    private PdfPTable buildProductTable(Notation notation) {
        Font headerFont = new Font(Font.HELVETICA, 9, Font.BOLD, Color.WHITE);
        PdfPTable productTable = new PdfPTable(2);
        productTable.setWidthPercentage(100);
        productTable.setSpacingAfter(12);
        productTable.setWidths(new float[]{2, 1});

        // En-têtes avec fond bleu foncé
        Stream.of("Rule ID", "Rule description").forEach(col -> {
            PdfPCell cell = new PdfPCell(new Phrase(col, headerFont));
            cell.setBackgroundColor(HEADER_BG);
            cell.setHorizontalAlignment(Element.ALIGN_CENTER);
            cell.setVerticalAlignment(Element.ALIGN_MIDDLE);
            cell.setPadding(6);
            productTable.addCell(cell);
        });

        // Données avec zébrage
        boolean alternate = false;
        for (NotationLine line : notation.getNotationLines()) {
            if ("CH_PRODUCT".equals(line.getCode())) {
                Optional<FieldConfiguration> fieldConfiguration = fieldConfigurationRepository.findByCode(line.getValue());
                String libelle = fieldConfiguration.map(FieldConfiguration::getLibelle).orElse(line.getValue());

                Color bgColor = alternate ? ZEBRA_COLOR : Color.WHITE;
                alternate = !alternate;

                productTable.addCell(createPdfCell(libelle != null ? libelle : "-", bgColor, 9));
                productTable.addCell(createPdfCell(line.getRisk() != null ? line.getRisk() : "-", bgColor, 9));
            }
        }

        return productTable;
    }

    /**
     * Crée une ligne info (label : valeur)
     */
    private Paragraph addInfoLine(String label, String value) {
        Paragraph p = new Paragraph();
        if (label != null && !label.isEmpty()) {
            p.add(new Chunk(label + ": ", LABEL_FONT));
        }
        p.add(new Chunk((value != null && !value.isEmpty()) ? value : "-", VALUE_FONT));
        p.setSpacingAfter(3);
        p.setAlignment(Element.ALIGN_LEFT);
        return p;
    }

    /**
     * Crée une cellule PDF stylisée
     */
    private PdfPCell createPdfCell(String text, Color bgColor, int fontSize) {
        Font cellFont = new Font(Font.HELVETICA, fontSize, Font.NORMAL, Color.BLACK);
        PdfPCell cell = new PdfPCell(new Phrase(text != null ? text : "-", cellFont));
        cell.setBackgroundColor(bgColor);
        cell.setPadding(5);
        cell.setHorizontalAlignment(Element.ALIGN_LEFT);
        cell.setVerticalAlignment(Element.ALIGN_MIDDLE);
        return cell;
    }

    /**
     * Récupère le code entité de l'utilisateur courant
     */
    private String getCurrentUserEntiteCode() {
        PortalUser user = currentUserService.getCurrentPortalUser();
        if (user == null || user.getEntite() == null || user.getEntite().getCode() == null) {
            throw new IllegalStateException("Utilisateur non authentifié ou sans entité/code");
        }
        return user.getEntite().getCode();
    }
}
