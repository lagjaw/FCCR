package com.sahambank.fccr.core.service.Impl;


import com.lowagie.text.*;
import com.lowagie.text.Font;
import com.lowagie.text.Image;
import com.lowagie.text.Rectangle;
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
import java.time.LocalDateTime;
import java.util.Optional;
import java.util.stream.Collectors;
import java.util.stream.Stream;

import static org.apache.poi.ss.util.CellUtil.createCell;

@Service
public class PdfServiceImpl  implements PdfService {


    private final NotationRepository notationRepository;
    private final SegmentRepository segmentRepository;
    private final FieldConfigurationRepository fieldConfigurationRepository;
    private final CurrentUserService currentUserService ;
    private final AuthorizationService authorizationService;
    NotationResponse dmnResp;


    public PdfServiceImpl(NotationRepository notationRepository, SegmentRepository segmentRepository,
                          FieldConfigurationRepository fieldConfigurationRepository,
                          CurrentUserService currentUserService,
                          AuthorizationService authorizationService
    ) {
        this.notationRepository = notationRepository;
        this.segmentRepository = segmentRepository;
        this.fieldConfigurationRepository = fieldConfigurationRepository;
        this.authorizationService = authorizationService ;
        this.currentUserService = currentUserService ;

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

        // Logo en haut de page
        writer.setPageEvent(new PdfPageEventHelper() {
            @Override
            public void onEndPage(PdfWriter writer, Document document) {
                PdfContentByte cb = writer.getDirectContent();
                try {
                    Image logo = Image.getInstance("src/main/resources/Assets/logo-SAHAM.jfif");
                    logo.scaleToFit(90, 90);
                    logo.setAbsolutePosition(document.leftMargin(), document.top() + 10);
                    cb.addImage(logo);
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        });

        document.open();

        // --- Titre principal ---
        Font titleFont = new Font(Font.HELVETICA, 18, Font.BOLD, new Color(10, 35, 66));
        Paragraph title = new Paragraph("Financial Crime Rating Report", titleFont);
        title.setAlignment(Element.ALIGN_CENTER);
        title.setSpacingAfter(20);
        document.add(title);

        // --- Pavé infos générales ---
        document.add(buildInfoTable(notation, currentUser));

        // --- Tableau des lignes de notation ---
        document.add(buildNotationTable(notation));

        // --- Tableau CH_PRODUCT ---
        document.add(buildProductTable(notation));

        document.close();
        return out.toByteArray();
    }

// ---------------- Méthodes utilitaires ----------------

    private PdfPTable buildInfoTable(Notation notation, PortalUser currentUser) {
        Font labelFont = new Font(Font.HELVETICA, 12, Font.BOLD, new Color(10, 35, 66));
        Font valueFont = new Font(Font.HELVETICA, 12, Font.NORMAL, Color.BLACK);

        PdfPTable infoTable = new PdfPTable(2);
        infoTable.setWidthPercentage(100);
        infoTable.setSpacingAfter(20);
        infoTable.setWidths(new float[]{1, 2});

        addInfoCellBox(infoTable, "Client :", notation.getClientCode(), labelFont, valueFont);
        addInfoCellBox(infoTable, "Segment :", notation.getSegmentCode(), labelFont , valueFont);
        addInfoCellBox(infoTable, "Date de création :", notation.getCreatedAt().toString(), labelFont, valueFont);

        String userName = currentUser.getFirstName() + " " + currentUser.getLastName();
        addInfoCellBox(infoTable, "Nom Utilisateur :", userName, labelFont, valueFont);

        String roles = currentUser.getRoles().stream()
                .map(role -> role.getCode().name())
                .collect(Collectors.joining(", "));
        addInfoCellBox(infoTable, "Roles :", roles, labelFont, valueFont);

        addInfoCellBox(infoTable, "Organisation :", getCurrentUserEntiteCode(), labelFont, valueFont);
        addInfoCellBox(infoTable, "Date et heure :", String.valueOf(notation.getCreatedAt()), labelFont, valueFont);
        addInfoCellBox(infoTable, "Note :", String.valueOf(notation.getNote()), labelFont, valueFont);
        addInfoCellBox(infoTable, "Flag ABC :", String.valueOf(notation.getFlagABC()), labelFont, valueFont);

        return infoTable;
    }

    private PdfPTable buildNotationTable(Notation notation) {
        Font headerFont = new Font(Font.HELVETICA, 12, Font.BOLD, Color.WHITE);
        PdfPTable table = new PdfPTable(7);
        table.setWidthPercentage(100);
        table.setWidths(new float[]{1.5f, 1.5f, 1.5f, 1.5f, 1, 1, 1});

        // En-têtes
        Stream.of("Code", "Libelle", "Valeur", "Risque", "Poids L", "Poids ML", "Poids MH").forEach(col -> {
            PdfPCell cell = new PdfPCell(new Phrase(col, headerFont));
            cell.setBackgroundColor(new Color(10, 35, 66));
            cell.setHorizontalAlignment(Element.ALIGN_CENTER);
            cell.setPadding(6);
            table.addCell(cell);
        });

        // Données avec zébrage
        boolean alternate = false;
        for (NotationLine line : notation.getNotationLines()) {
            Optional<FieldConfiguration> fieldConfiguration = fieldConfigurationRepository.findByCode(line.getCode());
            String libelle = fieldConfiguration.map(FieldConfiguration::getLibelle).orElse("N/A");

            Color bgColor = alternate ? new Color(245, 245, 245) : Color.WHITE;
            alternate = !alternate;

            table.addCell(createCell(line.getCode(), bgColor));
            table.addCell(createCell(libelle, bgColor));
            table.addCell(createCell(line.getValue(), bgColor));
            table.addCell(createCell(line.getRisk(), bgColor));
            table.addCell(createCell(String.valueOf(line.getWeightL()), bgColor));
            table.addCell(createCell(String.valueOf(line.getWeightMl()), bgColor));
            table.addCell(createCell(String.valueOf(line.getWeightMh()), bgColor));
        }

        return table;
    }

    private PdfPTable buildProductTable(Notation notation) {
        Font productHeaderFont = new Font(Font.HELVETICA, 12, Font.BOLD, Color.WHITE);
        PdfPTable productTable = new PdfPTable(2);
        productTable.setWidthPercentage(100);
        productTable.setSpacingAfter(20);
        productTable.setWidths(new float[]{2, 1});

        // En-têtes
        Stream.of("Produits", "Niveau De Risque").forEach(col -> {
            PdfPCell cell = new PdfPCell(new Phrase(col, productHeaderFont));
            cell.setBackgroundColor(new Color(10, 35, 66));
            cell.setHorizontalAlignment(Element.ALIGN_CENTER);
            cell.setPadding(6);
            productTable.addCell(cell);
        });

        // Données
        for (NotationLine line : notation.getNotationLines()) {
            if ("CH_PRODUCT".equals(line.getCode())) {
                Optional<FieldConfiguration> fieldConfiguration = fieldConfigurationRepository.findByCode(line.getValue());
                String libelle = fieldConfiguration.map(FieldConfiguration::getLibelle).orElse(line.getValue());

                productTable.addCell(new PdfPCell(new Phrase(libelle != null ? libelle : "-", new Font(Font.HELVETICA, 11))));
                productTable.addCell(new PdfPCell(new Phrase(line.getRisk() != null ? line.getRisk() : "-", new Font(Font.HELVETICA, 11))));
            }
        }

        return productTable;
    }

    String getCurrentUserEntiteCode() {
        PortalUser user = currentUserService.getCurrentPortalUser();
        if (user == null || user.getEntite() == null || user.getEntite().getCode() == null) {
            throw new IllegalStateException("Utilisateur non authentifié ou sans entité/code");
        }
        return user.getEntite().getCode();
    }

    private PdfPCell createCell(String text, Color bgColor) {
        PdfPCell cell = new PdfPCell(new Phrase(text != null ? text : "-", new Font(Font.HELVETICA, 11)));
        cell.setBackgroundColor(bgColor);
        cell.setPadding(5);
        return cell;
    }


    private void addInfoCellBox(PdfPTable table, String label, String value, Font labelFont, Font valueFont) {
        PdfPCell labelCell = new PdfPCell(new Phrase(label, labelFont));
        labelCell.setBackgroundColor(Color.GRAY); // fond gris pour distinguer les labels
        labelCell.setBorder(Rectangle.BOX);
        labelCell.setPadding(5);
        table.addCell(labelCell);

        PdfPCell valueCell = new PdfPCell(new Phrase(value != null ? value : "-", valueFont));
        valueCell.setBorder(Rectangle.BOX);
        valueCell.setPadding(5);
        table.addCell(valueCell);
    }

}
