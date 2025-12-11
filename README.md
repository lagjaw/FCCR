package com.sahambank.fccr.core.service.Impl;

import com.lowagie.text.*;
import com.lowagie.text.pdf.*;
import com.sahambank.fccr.core.dto.ConfigMoteurDetail.NotationResponse;
import com.sahambank.fccr.core.dto.UserInfo.Action;
import com.sahambank.fccr.core.dto.UserInfo.PortalRoles;
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
import java.time.LocalDateTime;
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
    private static final Font TITLE_FONT = new Font(Font.HELVETICA, 18, Font.BOLD, ACCENT_COLOR);
    private static final Font SECTION_TITLE_FONT = new Font(Font.HELVETICA, 12, Font.BOLD, ACCENT_COLOR);
    private static final Font LABEL_FONT = new Font(Font.HELVETICA, 11, Font.BOLD, PRIMARY_COLOR);
    private static final Font VALUE_FONT = new Font(Font.HELVETICA, 11, Font.NORMAL, Color.BLACK);
    private static final Font HEADER_FONT = new Font(Font.HELVETICA, 11, Font.BOLD, Color.WHITE);
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
        Document document = new Document(PageSize.A4, 50, 50, 70, 50);
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
                        logo.scaleToFit(90, 90);
                        logo.setAbsolutePosition(document.leftMargin(), document.top() + 10);
                        cb.addImage(logo);
                    }
                } catch (Exception e) {
                    // Log ou ignorer si le logo n'est pas trouvé
                }
                
                // Ligne de séparation en haut
                cb.setColorStroke(PRIMARY_COLOR);
                cb.setLineWidth(1f);
                cb.moveTo(document.leftMargin(), document.top() - 60);
                cb.lineTo(document.right() - document.rightMargin(), document.top() - 60);
                cb.stroke();
            }
        });

        document.open();

        // --- Badge "C2 - CONFIDENTIAL" ---
        Paragraph confidential = new Paragraph("C2 - CONFIDENTIAL", new Font(Font.HELVETICA, 10, Font.BOLD, PRIMARY_COLOR));
        confidential.setSpacingAfter(10);
        document.add(confidential);

        // --- Titre principal ---
        Paragraph title = new Paragraph("Financial Crime Rating Report", TITLE_FONT);
        title.setAlignment(Element.ALIGN_CENTER);
        title.setSpacingAfter(20);
        document.add(title);

        // --- Info Client (format texte structuré, 2 colonnes) ---
        document.add(buildClientInfoSection(notation, currentUser));

        // --- Section Rating Result (texte structuré) ---
        document.add(buildRatingResultSection(notation));

        // --- Section Rating Warnings (titre avec ligne) ---
        Paragraph warningsTitle = new Paragraph("Rating Warnings", SECTION_TITLE_FONT);
        warningsTitle.setSpacingBefore(15);
        warningsTitle.setSpacingAfter(10);
        document.add(warningsTitle);

        // --- Tableau des lignes de notation ---
        document.add(buildNotationTable(notation));

        // --- Section Applicable Business Rules ---
        Paragraph rulesTitle = new Paragraph("Applicable business rules", SECTION_TITLE_FONT);
        rulesTitle.setSpacingBefore(15);
        rulesTitle.setSpacingAfter(10);
        document.add(rulesTitle);

        // --- Tableau des règles ---
        document.add(buildProductTable(notation));

        // --- Pied de page ---
        Paragraph footer = new Paragraph("C2 - CONFIDENTIAL", new Font(Font.HELVETICA, 9, Font.NORMAL, PRIMARY_COLOR));
        footer.setAlignment(Element.ALIGN_RIGHT);
        footer.setSpacingBefore(20);
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
        mainTable.setSpacingAfter(20);
        mainTable.setWidths(new float[]{1, 1});
        mainTable.getDefaultCell().setBorder(Rectangle.NO_BORDER);
        mainTable.getDefaultCell().setPadding(5);

        // --- Colonne gauche ---
        PdfPCell leftCell = new PdfPCell();
        leftCell.setBorder(Rectangle.NO_BORDER);
        leftCell.setPadding(0);

        Paragraph leftContent = new Paragraph();
        leftContent.add(addInfoLine("Third party", ""));
        leftContent.add(addInfoLine("RCT Identifier", ""));
        leftContent.add(addInfoLine("Third party ID", notation.getClientCode()));
        leftContent.add(addInfoLine("Legal name", ""));
        leftContent.add(addInfoLine("Local Registration ID", ""));
        leftContent.add(addInfoLine("Ultimate Parent RCT", ""));
        leftContent.add(addInfoLine("Identifier", ""));
        leftContent.add(addInfoLine("Ultimate Parent Legal name", ""));
        leftContent.add(addInfoLine("Ultimate Parent's FCR", ""));
        leftContent.add(addInfoLine("Perimeter", ""));
        leftContent.add(addInfoLine("Countries of KYC", "MOROCCO (MA)"));
        leftContent.add(addInfoLine("agreements", ""));
        leftContent.add(addInfoLine("Third parties linked to same", ""));
        leftContent.add(addInfoLine("RCT/Country", ""));
        leftContent.add(addInfoLine("Third party Status", "Current"));
        leftContent.add(addInfoLine("Active Status", "Active"));
        leftContent.add(addInfoLine("Role", "Customer"));

        leftCell.addElement(leftContent);
        mainTable.addCell(leftCell);

        // --- Colonne droite ---
        PdfPCell rightCell = new PdfPCell();
        rightCell.setBorder(Rectangle.NO_BORDER);
        rightCell.setPadding(0);

        Paragraph rightContent = new Paragraph();
        rightContent.add(addInfoLine("Country", ""));
        rightContent.add(addInfoLine("Segment", notation.getSegmentCode()));
        rightContent.add(addInfoLine("LEI Code", ""));
        rightContent.add(addInfoLine("Ultimate Parent Country", ""));
        rightContent.add(addInfoLine("Ultimate Parent's Rating", ""));
        rightContent.add(addInfoLine("third party ID", ""));
        rightContent.add(addInfoLine("Ultimate Parent's Rating", ""));
        rightContent.add(addInfoLine("Date", ""));
        rightContent.add(addInfoLine("", ""));
        rightContent.add(addInfoLine("Date of End of Business", ""));
        rightContent.add(addInfoLine("Relationship", ""));
        rightContent.add(addInfoLine("Date of Active Status", notation.getCreatedAt() != null ? 
            notation.getCreatedAt().format(DATE_FORMATTER) : ""));

        rightCell.addElement(rightContent);
        mainTable.addCell(rightCell);

        return mainTable;
    }

    /**
     * Crée une ligne info (label : valeur)
     */
    private Paragraph addInfoLine(String label, String value) {
        Paragraph p = new Paragraph();
        if (!label.isEmpty()) {
            Chunk labelChunk = new Chunk(label + " ", LABEL_FONT);
            p.add(labelChunk);
        }
        if (value != null && !value.isEmpty()) {
            Chunk valueChunk = new Chunk(value, VALUE_FONT);
            p.add(valueChunk);
        } else {
            Chunk valueChunk = new Chunk("-", VALUE_FONT);
            p.add(valueChunk);
        }
        p.setSpacingAfter(4);
        p.setAlignment(Element.ALIGN_LEFT);
        return p;
    }

    /**
     * Construit la section Rating Result (texte structuré, 2 colonnes)
     */
    private PdfPTable buildRatingResultSection(Notation notation) {
        PdfPTable mainTable = new PdfPTable(2);
        mainTable.setWidthPercentage(100);
        mainTable.setSpacingAfter(20);
        mainTable.setWidths(new float[]{1, 1});
        mainTable.getDefaultCell().setBorder(Rectangle.NO_BORDER);
        mainTable.getDefaultCell().setPadding(5);

        // Titre de la section
        PdfPCell titleCell = new PdfPCell(new Phrase("Rating Result", SECTION_TITLE_FONT));
        titleCell.setColspan(2);
        titleCell.setBorder(Rectangle.NO_BORDER);
        mainTable.addCell(titleCell);

        // --- Colonne gauche ---
        PdfPCell leftCell = new PdfPCell();
        leftCell.setBorder(Rectangle.NO_BORDER);
        leftCell.setPadding(0);

        Paragraph leftContent = new Paragraph();
        leftContent.add(addInfoLine("Third Party ID rated", ""));
        leftContent.add(addInfoLine("Date, User name", notation.getCreatedAt() != null ? 
            notation.getCreatedAt().format(DATE_FORMATTER) : ""));
        leftContent.add(addInfoLine("Role, Organization", ""));
        leftContent.add(addInfoLine("Methodology", "V3 - Target"));

        leftCell.addElement(leftContent);
        mainTable.addCell(leftCell);

        // --- Colonne droite ---
        PdfPCell rightCell = new PdfPCell();
        rightCell.setBorder(Rectangle.NO_BORDER);
        rightCell.setPadding(0);

        Paragraph rightContent = new Paragraph();
        rightContent.add(addInfoLine("Computed Rating", notation.getNote() != null ? notation.getNote().toString() : ""));
        rightContent.add(addInfoLine("Model's Rating", notation.getNote() != null ? notation.getNote().toString() : ""));
        rightContent.add(addInfoLine("Rating Status", "Validated"));
        rightContent.add(addInfoLine("ABC Flag", notation.getFlagABC() != null ? notation.getFlagABC().toString() : ""));

        rightCell.addElement(rightContent);
        mainTable.addCell(rightCell);

        return mainTable;
    }

    /**
     * Construit le tableau des lignes de notation (Rating Warnings)
     */
    private PdfPTable buildNotationTable(Notation notation) {
        Font headerFont = new Font(Font.HELVETICA, 11, Font.BOLD, Color.WHITE);
        PdfPTable table = new PdfPTable(7);
        table.setWidthPercentage(100);
        table.setSpacingAfter(20);
        table.setWidths(new float[]{1.5f, 1.5f, 1.5f, 1.5f, 1, 1, 1});

        // En-têtes avec fond bleu foncé
        Stream.of("Code", "Libelle", "Valeur", "Risque", "Poids L", "Poids ML", "Poids MH").forEach(col -> {
            PdfPCell cell = new PdfPCell(new Phrase(col, headerFont));
            cell.setBackgroundColor(HEADER_BG);
            cell.setHorizontalAlignment(Element.ALIGN_CENTER);
            cell.setPadding(8);
            table.addCell(cell);
        });

        // Données avec zébrage
        boolean alternate = false;
        for (NotationLine line : notation.getNotationLines()) {
            Optional<FieldConfiguration> fieldConfiguration = fieldConfigurationRepository.findByCode(line.getCode());
            String libelle = fieldConfiguration.map(FieldConfiguration::getLibelle).orElse("N/A");

            Color bgColor = alternate ? ZEBRA_COLOR : Color.WHITE;
            alternate = !alternate;

            table.addCell(createPdfCell(line.getCode(), bgColor));
            table.addCell(createPdfCell(libelle, bgColor));
            table.addCell(createPdfCell(line.getValue(), bgColor));
            table.addCell(createPdfCell(line.getRisk(), bgColor));
            table.addCell(createPdfCell(String.valueOf(line.getWeightL()), bgColor));
            table.addCell(createPdfCell(String.valueOf(line.getWeightMl()), bgColor));
            table.addCell(createPdfCell(String.valueOf(line.getWeightMh()), bgColor));
        }

        return table;
    }

    /**
     * Construit le tableau des produits (Applicable business rules)
     */
    private PdfPTable buildProductTable(Notation notation) {
        Font headerFont = new Font(Font.HELVETICA, 11, Font.BOLD, Color.WHITE);
        PdfPTable productTable = new PdfPTable(2);
        productTable.setWidthPercentage(100);
        productTable.setSpacingAfter(20);
        productTable.setWidths(new float[]{2, 1});

        // En-têtes avec fond bleu foncé
        Stream.of("Rule ID", "Rule description").forEach(col -> {
            PdfPCell cell = new PdfPCell(new Phrase(col, headerFont));
            cell.setBackgroundColor(HEADER_BG);
            cell.setHorizontalAlignment(Element.ALIGN_CENTER);
            cell.setPadding(8);
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

                productTable.addCell(createPdfCell(libelle != null ? libelle : "-", bgColor));
                productTable.addCell(createPdfCell(line.getRisk() != null ? line.getRisk() : "-", bgColor));
            }
        }

        return productTable;
    }

    /**
     * Crée une cellule PDF stylisée
     */
    private PdfPCell createPdfCell(String text, Color bgColor) {
        PdfPCell cell = new PdfPCell(new Phrase(text != null ? text : "-", VALUE_FONT));
        cell.setBackgroundColor(bgColor);
        cell.setPadding(6);
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
