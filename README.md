# FCCR
import { HttpClient } from "@angular/common/http";
import { Injectable } from "@angular/core";
import { Observable } from "rxjs";
import { environment } from "src/environments/environment";



@Injectable({
  providedIn: 'root'
})
export class NotationService {

     private apiUrl = environment.api.url
  constructor(private http: HttpClient) {
  }
  getSegmentById(id: number): Observable<any> {
    return this.http.get<any>(`${this.apiUrl}segments/${id}`);
  }
    saveNotation(payload: any): Observable<any> {
    return this.http.post<any>(`${this.apiUrl}notations`, payload);
  }
}

---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

<nb-card accent="primary" class="segment-card">
  <nb-card-header class="d-flex flex-row justify-content-between align-items-center">
    <h5 class="title-animation title-heading text-uppercase my-auto p-2">Recherche notation</h5>
  </nb-card-header>

  <nb-card-body>
    <form [formGroup]="fieldForm" (ngSubmit)="submit()">
      <nb-form-field fullWidth>
        <nb-select
          fullWidth
          placeholder="Sélectionner un segment"
          formControlName="segment"
          (selectedChange)="onSegementChange($event)">
          <nb-option *ngFor="let type of referentiels$" [value]="type.id">
            {{ type.libelle }}
          </nb-option>
        </nb-select>
      </nb-form-field>
      <small class="error" *ngIf="fieldForm.get('segment')?.invalid && fieldForm.get('segment')?.touched">
        Ce champ est requis.
      </small>

      <ng-container *ngFor="let area of segement.areas">
        <nb-card class="area-card">
          <nb-card-body class="area-grid">
            <div *ngFor="let config of area.fieldConfigurations" class="field-row">
              <div *ngIf="config.searchable">
                <label><strong>{{ config.libelle }}</strong></label>
                <div class="form-line">
                  <nb-form-field *ngIf="config.type === 'select'">
                    <nb-select formControlName="{{ config.code }}" multiple>
                      <nb-option
                        *ngFor="let item of config.risqueValueList"
                        [value]="item.listValueItem?.id">
                        {{ item.listValueItem?.libelle }}
                      </nb-option>
                    </nb-select>
                  </nb-form-field>

                  <nb-form-field *ngIf="config.type === 'boolean'">
                    <nb-select formControlName="{{ config.code }}">
                      <nb-option [value]="true">Vrai</nb-option>
                      <nb-option [value]="false">Faux</nb-option>
                    </nb-select>
                  </nb-form-field>

                  <nb-form-field *ngIf="config.type === 'date'">
                    <input
                      nbInput
                      type="date"
                      formControlName="{{ config.code }}"
                      placeholder="Choisir une date" />
                  </nb-form-field>

                  <nb-form-field *ngIf="config.type === 'text' || config.type === 'number'">
                    <input
                      nbInput
                      [type]="config.type"
                      formControlName="{{ config.code }}"
                      placeholder="Saisir une valeur" />
                  </nb-form-field>
                </div>
              </div>
            </div>
          </nb-card-body>
        </nb-card>
      </ng-container>

      <div class="text-center mt-4">
        <button nbButton status="primary" type="submit">
          <nb-icon icon="cloud-upload-outline"></nb-icon> Rechercher
        </button>
      </div>
    </form>
  </nb-card-body>
</nb-card>

<nb-card class="main-card" *ngIf="search">
  <nb-card-header>Résultat de recherche</nb-card-header>
  <nb-card-body>
    <div class="result-row"><strong>Code Client :</strong> {{ recherche.codeClient }}</div>
    <div class="result-row"><strong>Nom / RC :</strong> {{ recherche.nomPrenomOuRC }}</div>
    <div class="result-row"><strong>Note :</strong> {{ recherche.note }}</div>
    <div class="result-row"><strong>Dernière note :</strong> {{ recherche?.dateNote }}</div>
  </nb-card-body>
</nb-card>
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------import { Component, OnInit } from '@angular/core';
import { FormArray, FormBuilder, FormGroup, Validators } from '@angular/forms';
import { Referentiel } from 'src/app/core/models/Referentiel';
import { ReferentielTypes } from '../traitement-demande/model/fieldConfig';
import { NbToastrService } from '@nebular/theme';
import { GestionReferentielsService } from '../gestion-referentiel/services/gestion-referentiels.service';
import { SegmentDtoVues } from '../notation/model/segementDtoVue';
import { NotationService } from '../notation/notation.service';

@Component({
  selector: 'app-client',
  standalone: false,
  templateUrl: './client.component.html',
  styleUrl: './client.component.scss'
})
export class ClientComponent implements OnInit{


    fieldForm!: FormGroup;
    fieldGlobal:any[]=[]
    selectedRef: string = "segments";
    referentiels$ !: Referentiel[];
    referentielTypes: ReferentielTypes[] = [];
     segement: SegmentDtoVues = {} as SegmentDtoVues;
    search=false;
    resultatRecherche: any;

    recherche: {
  codeClient?: string;
  nomPrenomOuRC?: string;
  note?: number;
  dateNote?: string;
} = {};

  constructor(private fb: FormBuilder,private toasterService:NbToastrService,private serviceRef: GestionReferentielsService,private notationService:NotationService) {

      this.referentielTypes  = [
      {value:'segments', label:'Segment', hasExpression:false},
      {value:'areas', label:'Area', hasExpression:false},
      {value:'listValues', label:'List Value ', hasExpression:true},
      {value:'listValueItems', label:'List Value Item', hasExpression:false},
      {value:'risqueValueLists', label:'Risque Value', hasExpression:false},
      {value:'risqueValueItems', label:'Risque Value Item', hasExpression:false},
    ];

  }





  ngOnInit(): void {
       this.serviceRef.getReferentiel(this.selectedRef).subscribe(data => {
        this.referentiels$ = data
      })
          this.initialiserChampsDynamiques();
  }



  get fieldConfigs(): FormArray {
  return this.fieldForm.get('fields') as FormArray;
}
get fieldsControls(): FormGroup[] {
  return (this.fieldForm.get('fields') as FormArray).controls as FormGroup[];
}


fieldsArray(): FormArray {
  return this.fieldForm.get('fields') as FormArray;
}

 initialiserChampsDynamiques(): void {
    const group: { [key: string]: any } = {};

    this.fieldGlobal.forEach(config => {
      group[config.code] = [null];
    });

    this.fieldForm = this.fb.group(group);
  }

  envoyerFormulaire(): void {
    const payload = this.fieldForm.value;
    console.log(payload);
    this.search = true
      const donnees = this.fieldForm.value;
   this.recherche = {
    codeClient: donnees.identifiantClient,
    nomPrenomOuRC: donnees.Nom ,
    note: 4.5,
    dateNote: '2025-03-12'
  };
  }


  submit(): void {
    this.search=true
    console.log(this.fieldForm);

  }

  onSegementChange(event: any) {
  const selectedId = event?.id || null;
  this.fieldForm.patchValue({ valueId: selectedId });
  this.notationService.getSegmentById(event).subscribe({
        next: (response) => {
          response.areas.forEach((e:any)=>{
            console.log("dsdsqdq",e.fieldConfigurations);

              e.fieldConfigurations.forEach((f:any)=>{
                console.log("rrr",f);

                  if(f.searchable){
                    this.fieldGlobal.push(f)
                    console.log(this.fieldGlobal);

                    this.initialiserFormulaire();

                  }
              })
          })

          this.segement= response;


        },
        error: (err:any) => {
          console.error('Erreur lors de la création du champ :', err);
        }
      });
    }

  initialiserFormulaire(): void {
  const group: { [key: string]: any } = {};

  this.fieldGlobal.forEach(config => {
    const validators = [];
    if (config.code === 'RC') {
      validators.push(Validators.required);
      validators.push(Validators.minLength(5));
    }
    group[config.code] = [null, validators];
  });

  this.fieldForm = this.fb.group(group);
  console.log('🧾 Champs du formulaire :', Object.keys(this.fieldForm.controls));
}



}
--------------------------------------------------------------------------------import { Component, OnInit } from '@angular/core';
import { FormArray, FormBuilder, FormGroup, Validators } from '@angular/forms';
import { Referentiel } from 'src/app/core/models/Referentiel';
import { ReferentielTypes } from '../traitement-demande/model/fieldConfig';
import { NbToastrService } from '@nebular/theme';
import { GestionReferentielsService } from '../gestion-referentiel/services/gestion-referentiels.service';
import { SegmentDtoVues } from '../notation/model/segementDtoVue';
import { NotationService } from '../notation/notation.service';

@Component({
  selector: 'app-client',
  standalone: false,
  templateUrl: './client.component.html',
  styleUrl: './client.component.scss'
})
export class ClientComponent implements OnInit{


    fieldForm!: FormGroup;
    fieldGlobal:any[]=[]
    selectedRef: string = "segments";
    referentiels$ !: Referentiel[];
    referentielTypes: ReferentielTypes[] = [];
     segement: SegmentDtoVues = {} as SegmentDtoVues;
    search=false;
    resultatRecherche: any;

    recherche: {
  codeClient?: string;
  nomPrenomOuRC?: string;
  note?: number;
  dateNote?: string;
} = {};

  constructor(private fb: FormBuilder,private toasterService:NbToastrService,private serviceRef: GestionReferentielsService,private notationService:NotationService) {

      this.referentielTypes  = [
      {value:'segments', label:'Segment', hasExpression:false},
      {value:'areas', label:'Area', hasExpression:false},
      {value:'listValues', label:'List Value ', hasExpression:true},
      {value:'listValueItems', label:'List Value Item', hasExpression:false},
      {value:'risqueValueLists', label:'Risque Value', hasExpression:false},
      {value:'risqueValueItems', label:'Risque Value Item', hasExpression:false},
    ];

  }





  ngOnInit(): void {
       this.serviceRef.getReferentiel(this.selectedRef).subscribe(data => {
        this.referentiels$ = data
      })
          this.initialiserChampsDynamiques();
  }



  get fieldConfigs(): FormArray {
  return this.fieldForm.get('fields') as FormArray;
}
get fieldsControls(): FormGroup[] {
  return (this.fieldForm.get('fields') as FormArray).controls as FormGroup[];
}


fieldsArray(): FormArray {
  return this.fieldForm.get('fields') as FormArray;
}

 initialiserChampsDynamiques(): void {
    const group: { [key: string]: any } = {};

    this.fieldGlobal.forEach(config => {
      group[config.code] = [null];
    });

    this.fieldForm = this.fb.group(group);
  }

  envoyerFormulaire(): void {
    const payload = this.fieldForm.value;
    console.log(payload);
    this.search = true
      const donnees = this.fieldForm.value;
   this.recherche = {
    codeClient: donnees.identifiantClient,
    nomPrenomOuRC: donnees.Nom ,
    note: 4.5,
    dateNote: '2025-03-12'
  };
  }


  submit(): void {
    this.search=true
    console.log(this.fieldForm);

  }

  onSegementChange(event: any) {
  const selectedId = event?.id || null;
  this.fieldForm.patchValue({ valueId: selectedId });
  this.notationService.getSegmentById(event).subscribe({
        next: (response) => {
          response.areas.forEach((e:any)=>{
            console.log("dsdsqdq",e.fieldConfigurations);

              e.fieldConfigurations.forEach((f:any)=>{
                console.log("rrr",f);

                  if(f.searchable){
                    this.fieldGlobal.push(f)
                    console.log(this.fieldGlobal);

                    this.initialiserFormulaire();

                  }
              })
          })

          this.segement= response;


        },
        error: (err:any) => {
          console.error('Erreur lors de la création du champ :', err);
        }
      });
    }

  initialiserFormulaire(): void {
  const group: { [key: string]: any } = {};

  this.fieldGlobal.forEach(config => {
    const validators = [];
    if (config.code === 'RC') {
      validators.push(Validators.required);
      validators.push(Validators.minLength(5));
    }
    group[config.code] = [null, validators];
  });

  this.fieldForm = this.fb.group(group);
  console.log('🧾 Champs du formulaire :', Object.keys(this.fieldForm.controls));
}



}
------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------package com.sahambank.fccr.core.service.Impl;

import com.sahambank.fccr.core.dto.*;
import com.sahambank.fccr.core.dto.segment.SegmentSaveDto;
import com.sahambank.fccr.core.entities.*;
import com.sahambank.fccr.core.entities.file.ProductData;
import com.sahambank.fccr.core.mapper.RisqueEntryMapper;
import com.sahambank.fccr.core.repository.ItemRepository;
import com.sahambank.fccr.core.repository.NotationRepository;
import com.sahambank.fccr.core.repository.ProductDataRepository;
import com.sahambank.fccr.core.repository.RisqueEntryRepository;
import com.sahambank.fccr.core.service.NotationService;
import jakarta.transaction.Transactional;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.stereotype.Service;


import java.time.YearMonth;
import java.util.*;
import java.util.stream.Collectors;

@Service
public class NotationServiceImpl implements NotationService {

    private final NotationRepository notationRepository;
    private final ItemRepository itemRepository;
    private final ProductDataRepository productDataRepository;
    private final RisqueEntryRepository risqueEntryRepository;
    private final RisqueEntryMapper risqueEntryMapper;
    private final CurrentUserService currentUserService;

    public NotationServiceImpl(NotationRepository notationRepository,
                               ItemRepository itemRepository,
                               ProductDataRepository productDataRepository,
                               RisqueEntryRepository risqueEntryRepository,
                               RisqueEntryMapper risqueEntryMapper,
                               CurrentUserService currentUserService) {
        this.notationRepository = notationRepository;
        this.itemRepository = itemRepository;
        this.productDataRepository = productDataRepository;
        this.risqueEntryRepository = risqueEntryRepository;
        this.risqueEntryMapper = risqueEntryMapper;
        this.currentUserService=currentUserService;
    }

    private static boolean isBlank(String s) {
        return s == null || s.trim().isEmpty();
    }

    // Ne jette plus d’exception : renvoie Optional pour permettre le fallback
    private Optional<ProductData> resolveOptionalProductData(String clientCode) {
        String currentMonth = YearMonth.now().toString(); // YYYY-MM
        return productDataRepository
                .findByRctIdentifierClientAndCollectMonth(clientCode, currentMonth)
                .or(() -> productDataRepository.findTopByRctIdentifierClientOrderByCollectMonthDesc(clientCode));
    }

    // Charge l’historique ProductData sur 12 mois (si disponible), puis récupère les risques liés au client
    // et mappe en DTO. On garde ça simple côté requêtes.
    private List<RisqueEntryDto> loadBackendRiskDtosForClientLast12Months(String clientCode) {
        String sinceMonth = YearMonth.now().minusMonths(11).toString();
        List<ProductData> historiques = productDataRepository.findAllSinceMonth(clientCode, sinceMonth);
        if (historiques == null || historiques.isEmpty()) {
            return Collections.emptyList();
        }
        // La requête findFirstByRctIdentifierClient renvoie les risques du client (nom mal choisi)
        // On les mappe une seule fois pour éviter les doublons par itérations.
        List<RisqueEntry> risques = risqueEntryRepository.findFirstByRctIdentifierClient(clientCode);
        if (risques == null || risques.isEmpty()) {
            return Collections.emptyList();
        }
        return risques.stream()
                .map(risqueEntryMapper::fromRisqueEntryToDTO)
                .filter(Objects::nonNull)
                .toList();
    }

    // Clé de dédoublonnage
    private String keyOf(RisqueEntryDto r) {
        String listCode = (r.getListValueItem() != null && r.getListValueItem().getCode() != null)
                ? r.getListValueItem().getCode() : "";
        String riskCode = (r.getRisqueValueItem() != null && r.getRisqueValueItem().getCode() != null)
                ? r.getRisqueValueItem().getCode() : "";
        return listCode + "|" + riskCode;
    }

    // Fusion sans doublons. Priorité "back" (données produit) par défaut, puis "front".
    private List<RisqueEntryDto> mergeFrontBackDistinct(List<RisqueEntryDto> front, List<RisqueEntryDto> back) {
        Map<String, RisqueEntryDto> unique = new LinkedHashMap<>();
        if (back != null) {
            for (RisqueEntryDto r : back) {
                if (r != null) unique.put(keyOf(r), r);
            }
        }
        if (front != null) {
            for (RisqueEntryDto r : front) {
                if (r != null) unique.putIfAbsent(keyOf(r), r);
            }
        }
        return new ArrayList<>(unique.values());
    }

    // Transforme les DTOs en entités RisqueEntry et relie au fieldValue
    private List<RisqueEntry> mapToEntities(List<RisqueEntryDto> dtos, NotationFieldValue fv) {
        if (dtos == null) return Collections.emptyList();
        return dtos.stream().map(r -> {
            RisqueEntry emb = new RisqueEntry();
            if (r.getListValueItem() != null && !isBlank(r.getListValueItem().getCode())) {
                emb.setListValueItem(resolveItemByCodeOrCreate(r.getListValueItem().getCode(), r.getListValueItem().getLibelle()));
            }
            if (r.getRisqueValueItem() != null && !isBlank(r.getRisqueValueItem().getCode())) {
                emb.setRisqueValueItem(resolveItemByCodeOrCreate(r.getRisqueValueItem().getCode(), r.getRisqueValueItem().getLibelle()));
            }
            emb.setNotationFieldValue(fv);
            return emb;
        }).collect(Collectors.toList());
    }

    private Item resolveItemByCodeOrCreate(String code, String libelle) {
        return itemRepository.findByCode(code).orElseGet(() -> {
            Item item = new Item();
            item.setCode(code);
            item.setLibelle(libelle);
            return itemRepository.save(item);
        });
    }
    @Transactional
    @Override
    public NotationDto save(NotationDto dto) {
        if (dto == null) throw new IllegalArgumentException("NotationDto is required");
        if (dto.getSEGMENT() == null || isBlank(dto.getSEGMENT().getCode()))
            throw new IllegalArgumentException("SEGMENT and SEGMENT.code are required");
        if (isBlank(dto.getClientCode()))
            throw new IllegalArgumentException("clientCode is required");

        // Récupère éventuellement le ProductData associé
        Optional<ProductData> productDataOpt = resolveOptionalProductData(dto.getClientCode());

        // Création de l'entité Notation
        Notation notation = new Notation();
        notation.setSegmentCode(dto.getSEGMENT().getCode());
        notation.setClientCode(dto.getClientCode());
        notation.setProductData(productDataOpt.orElse(null));

        List<NotationFieldValue> fieldValues = new ArrayList<>();
        List<Integer> scores = new ArrayList<>();

        if (dto.getFieldConfigurations() != null) {
            List<RisqueEntryDto> backendDtosCache = null;

            for (FiledValueDto f : dto.getFieldConfigurations()) {
                if (f == null) continue;

                NotationFieldValue fv = new NotationFieldValue();
                fv.setFieldConfigurationId(f.getId());
                fv.setFieldCode(f.getCode());
                fv.setFonction(f.getFonction());
                fv.setFonctiontype(f.getFonctionType());
                fv.setNotation(notation);

                List<RisqueEntry> risqueEntries;

                boolean isProduitFn = Boolean.TRUE.equals(f.getFonction())
                        && "Produit".equalsIgnoreCase(f.getFonctionType());

                if (isProduitFn) {
                    if (backendDtosCache == null) {
                        backendDtosCache = loadBackendRiskDtosForClientLast12Months(dto.getClientCode());
                    }
                    List<RisqueEntryDto> frontDtos = (f.getRisqueValueList() != null) ? f.getRisqueValueList() : Collections.emptyList();
                    List<RisqueEntryDto> fusion = mergeFrontBackDistinct(frontDtos, backendDtosCache);
                    risqueEntries = mapToEntities(fusion, fv);

                    // Ajout des scores pour chaque risque fusionné
                    for (RisqueEntryDto r : fusion) {
                        if (r != null && r.getRisqueValueItem() != null && r.getRisqueValueItem().getCode() != null) {
                            scores.add(mapRiskLevelToScore(r.getRisqueValueItem().getCode()));
                        }
                    }

                } else {
                    List<RisqueEntryDto> frontDtos = (f.getRisqueValueList() != null) ? f.getRisqueValueList() : Collections.emptyList();
                    Map<String, RisqueEntryDto> unique = new LinkedHashMap<>();
                    for (RisqueEntryDto r : frontDtos) {
                        if (r != null) unique.put(keyOf(r), r);

                        if (r != null && r.getRisqueValueItem() != null &&
                                r.getRisqueValueItem().getCode() != null) {
                            scores.add(mapRiskLevelToScore(r.getRisqueValueItem().getCode()));
                        }
                    }
                    risqueEntries = mapToEntities(new ArrayList<>(unique.values()), fv);
                }

                fv.setRisqueValueList(risqueEntries);
                fieldValues.add(fv);
            }
        }

        notation.setFieldValues(fieldValues);

        // ➡ Calcul de la moyenne des scores si on en a
        if (!scores.isEmpty()) {
            double moyenne = scores.stream()
                    .mapToInt(Integer::intValue)
                    .average()
                    .orElse(0);
            notation.setNote(moyenne);
        }

        PortalUser currentUser = currentUserService.getCurrentPortalUser();
        notation.setCreatedBy(currentUser.getEmail());

        // Lier à ProductData si présent
        productDataOpt.ifPresent(pd -> pd.getNotations().add(notation));

        // Persistance (cascade active pour FieldValues et RisqueEntries)
        notationRepository.save(notation);

        // Retourne un DTO complet grâce au mapper
        return mapToDto(notation);
    }

    private int mapRiskLevelToScore(String level) {
        return switch (level.toUpperCase()) {
            case "HIGH" -> 3;
            case "MID", "MEDIUM" -> 2;
            case "LOW" -> 1;
            default -> 0;
        };
    }

    @org.springframework.transaction.annotation.Transactional(readOnly = true)
    @Override
    public NotationDto simulate(NotationDto dto) {
        if (dto == null) throw new IllegalArgumentException("NotationDto is required");
        if (dto.getSEGMENT() == null || isBlank(dto.getSEGMENT().getCode()))
            throw new IllegalArgumentException("SEGMENT and SEGMENT.code are required");
        if (isBlank(dto.getClientCode()))
            throw new IllegalArgumentException("clientCode is required");

        // On réutilise la logique de calcul
        CalculationResult result = calculerNoteEtChamps(dto, true);
        Notation notation = result.notation();

        // On retourne un DTO "virtuel" (pas persisté)
        NotationDto out = mapToDto(notation);
        out.setClientCode(null);
        out.setCreatedBy(null);

        return out;
    }


    @Override
    public Map<String, Item> mapToListValue(String clientCode) {
        Map<String, Item> resultMap = new HashMap<>();
        Optional<ProductData> productDataOpt = productDataRepository.findFirstByRctIdentifierClient(clientCode);
        if (productDataOpt.isEmpty()) {
            return resultMap;
        }
        ProductData productData = productDataOpt.get();
        // Attention: cette méthode réduit à un seul Item par client; si besoin, retourner une liste
        for (Notation n : productData.getNotations()) {
            for (NotationFieldValue fv : n.getFieldValues()) {
                for (RisqueEntry re : fv.getRisqueValueList()) {
                    resultMap.put(productData.getRctIdentifierClient(), re.getRisqueValueItem());
                }
            }
        }
        return resultMap;
    }

    private NotationDto mapToDto(Notation notation) {
        if (notation == null) return null;

        NotationDto dto = new NotationDto();

        // SEGMENT
        SegmentSaveDto segmentDto = new SegmentSaveDto();
        segmentDto.setCode(notation.getSegmentCode());
        dto.setSEGMENT(segmentDto);

        // Client Code
        dto.setClientCode(notation.getClientCode());

        // FieldConfigurations
        if (notation.getFieldValues() != null && !notation.getFieldValues().isEmpty()) {
            List<FiledValueDto> fieldDtos = notation.getFieldValues().stream().map(fv -> {
                FiledValueDto fvd = new FiledValueDto();
                fvd.setId(fv.getFieldConfigurationId());
                fvd.setCode(fv.getFieldCode());
                fvd.setFonction(fv.getFonction());
                fvd.setFonctionType(fv.getFonctiontype());

                // RisqueValueList
                if (fv.getRisqueValueList() != null && !fv.getRisqueValueList().isEmpty()) {
                    List<RisqueEntryDto> risqueDtos = fv.getRisqueValueList().stream().map(re -> {
                        RisqueEntryDto reDto = new RisqueEntryDto();

                        if (re.getListValueItem() != null) {
                            ItemDto listItem = new ItemDto();
                            listItem.setCode(re.getListValueItem().getCode());
                            listItem.setLibelle(re.getListValueItem().getLibelle());
                            reDto.setListValueItem(listItem);
                        }

                        if (re.getRisqueValueItem() != null) {
                            ItemDto riskItem = new ItemDto();
                            riskItem.setCode(re.getRisqueValueItem().getCode());
                            riskItem.setLibelle(re.getRisqueValueItem().getLibelle());
                            reDto.setRisqueValueItem(riskItem);
                        }

                        return reDto;
                    }).toList();
                    fvd.setRisqueValueList(risqueDtos);
                }

                return fvd;
            }).toList();

            dto.setFieldConfigurations(fieldDtos);
        }

        return dto;
    }


    @Override
    public List<RiskPairDto> getUniqueRisks(String clientCode, String productCode) {
        List<RisqueEntry> entries = risqueEntryRepository.findByClientAndProduct(clientCode, productCode);
        Map<String, RiskPairDto> unique = new LinkedHashMap<>();
        for (RisqueEntry e : entries) {
            Item list = e.getListValueItem();
            Item risk = e.getRisqueValueItem();

            String listCode = list != null ? list.getCode() : "";
            String riskCode = risk != null ? risk.getCode() : "";
            String key = listCode + "|" + riskCode;

            if (!unique.containsKey(key)) {
                ItemDto listDto = new ItemDto();
                if (list != null) {
                    listDto.setCode(list.getCode());
                    listDto.setLibelle(list.getLibelle());
                }
                ItemDto riskDto = new ItemDto();
                if (risk != null) {
                    riskDto.setCode(risk.getCode());
                    riskDto.setLibelle(risk.getLibelle());
                }
                unique.put(key, new RiskPairDto(listDto, riskDto));
            }
        }
        return new ArrayList<>(unique.values());
    }

    private CalculationResult calculerNoteEtChamps(NotationDto dto, boolean chargerBackend) {
        List<NotationFieldValue> fieldValues = new ArrayList<>();
        List<Integer> scores = new ArrayList<>();
        List<RisqueEntryDto> backendDtosCache = null;

        Notation notation = new Notation();
        notation.setSegmentCode(dto.getSEGMENT().getCode());
        notation.setClientCode(dto.getClientCode());

        if (dto.getFieldConfigurations() != null) {
            for (FiledValueDto f : dto.getFieldConfigurations()) {
                if (f == null) continue;

                NotationFieldValue fv = new NotationFieldValue();
                fv.setFieldConfigurationId(f.getId());
                fv.setFieldCode(f.getCode());
                fv.setFonction(f.getFonction());
                fv.setFonctiontype(f.getFonctionType());
                fv.setNotation(notation);

                List<RisqueEntry> risqueEntries;
                boolean isProduitFn = Boolean.TRUE.equals(f.getFonction())
                        && "Produit".equalsIgnoreCase(f.getFonctionType());

                if (isProduitFn && chargerBackend) {
                    if (backendDtosCache == null) {
                        backendDtosCache = loadBackendRiskDtosForClientLast12Months(dto.getClientCode());
                    }
                    List<RisqueEntryDto> frontDtos = (f.getRisqueValueList() != null) ? f.getRisqueValueList() : Collections.emptyList();
                    List<RisqueEntryDto> fusion = mergeFrontBackDistinct(frontDtos, backendDtosCache);
                    risqueEntries = mapToEntities(fusion, fv);

                    // Remonter les scores depuis RisqueValueItem
                    for (RisqueEntryDto r : fusion) {
                        if (r != null && r.getRisqueValueItem() != null && r.getRisqueValueItem().getCode() != null) {
                            scores.add(mapRiskLevelToScore(r.getRisqueValueItem().getCode()));
                        }
                    }
                } else {
                    List<RisqueEntryDto> frontDtos = (f.getRisqueValueList() != null) ? f.getRisqueValueList() : Collections.emptyList();
                    Map<String, RisqueEntryDto> unique = new LinkedHashMap<>();
                    for (RisqueEntryDto r : frontDtos) {
                        if (r != null) unique.put(keyOf(r), r);

                        if (r != null && r.getRisqueValueItem() != null &&
                                r.getRisqueValueItem().getCode() != null) {
                            scores.add(mapRiskLevelToScore(r.getRisqueValueItem().getCode()));
                        }
                    }
                    risqueEntries = mapToEntities(new ArrayList<>(unique.values()), fv);
                }

                fv.setRisqueValueList(risqueEntries);
                fieldValues.add(fv);
            }
        }

        double moyenne = scores.isEmpty() ? 0.0 :
                scores.stream().mapToInt(Integer::intValue).average().orElse(0);

        notation.setFieldValues(fieldValues);
        notation.setNote(moyenne);

        return new CalculationResult(notation, moyenne);
    }

    private record CalculationResult(Notation notation, double note) {}

    @org.springframework.transaction.annotation.Transactional(readOnly = true)
    public List<NotationDto> searchByAreas(List<Long> areaIds) {
        return notationRepository.findByAreaIds(areaIds)
                .stream()
                .map(this::mapToDto)
                .toList();
    }



}

------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------package com.sahambank.fccr.core.controller;

import com.sahambank.fccr.core.dto.NotationDto;
import com.sahambank.fccr.core.entities.Item;
import com.sahambank.fccr.core.entities.file.ProductData;
import com.sahambank.fccr.core.service.NotationService;
import jakarta.validation.Valid;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.List;
import java.util.Map;

@RestController
@RequestMapping("notations")
public class NotationController {

    private final NotationService notationService;

    public NotationController(NotationService notationService) {
        this.notationService = notationService;
    }

    @PostMapping
    public ResponseEntity<NotationDto> save(@RequestBody NotationDto dto) {
        NotationDto saved = notationService.save(dto);
        return ResponseEntity.ok(saved);
    }


    @GetMapping
    public Map<String, Item> save(@RequestParam String s) {
        return notationService.mapToListValue(s);
      //  return ResponseEntity.ok(Map.of("id", id, "message", "Notation enregistrée"));
    }

    @PostMapping("/simulate")
    public ResponseEntity<NotationDto> simulate(@RequestBody @Valid NotationDto dto) {
        return ResponseEntity.ok(notationService.simulate(dto));
    }

    @GetMapping("/search")
    public ResponseEntity<List<NotationDto>> searchByAreas(@RequestParam List<Long> areaIds) {
        return ResponseEntity.ok(notationService.searchByAreas(areaIds));
    }



}

----------------------------------------------------------------------------------------------------package com.sahambank.fccr.core.dto;

import com.fasterxml.jackson.annotation.JsonProperty;
import com.sahambank.fccr.core.dto.segment.SegmentSaveDto;
import lombok.AllArgsConstructor;
import lombok.Getter;
import lombok.NoArgsConstructor;
import lombok.Setter;

import java.time.LocalDateTime;
import java.util.List;
@AllArgsConstructor
@NoArgsConstructor
@Getter
@Setter
public class NotationDto {

    Long id;
    @JsonProperty("SEGMENT")
    private SegmentSaveDto SEGMENT;
    private List<FiledValueDto> fieldConfigurations;
    private String clientCode;

    private String createdBy ;

    LocalDateTime createdAt ;

    private double note ;


}
----------------------------------------------------------------------------------------------------package com.sahambank.fccr.core.repository;

import com.sahambank.fccr.core.entities.Notation;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;

import java.util.List;

public interface NotationRepository extends JpaRepository<Notation, Long> {
    @Query("""
        SELECT n FROM Notation n
        JOIN Segment s ON s.code = n.segmentCode
        JOIN s.areas a
        WHERE a.id IN :areaIds
    """)
    List<Notation> findByAreaIds(@Param("areaIds") List<Long> areaIds);
}
