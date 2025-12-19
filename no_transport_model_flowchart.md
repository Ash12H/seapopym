# NoTransportModel - Flowchart

Ce diagramme représente le flux de données et de calcul du modèle NoTransportModel avec le kernel NoTransportKernel.

```mermaid
flowchart TB
    %% ===== INPUTS: Forcings =====
    F1[temperature]
    F2[primary_production]
    F3[initial_condition_production]
    F4[initial_condition_biomass]

    %% ===== PARAMETERS =====
    P1[energy_transfert]
    P2[lambda_temperature_0]
    P3[gamma_lambda_temperature]
    P4[tr_0]
    P5[gamma_tr]
    P6[day_layer]
    P7[night_layer]
    P8[timesteps_number]
    P9[mean_timestep]
    P10[angle_horizon_sun]
    P13[timestep]

    %% ===== KERNEL UNITS =====

    %% 1. TemperatureGillooly
    F1 --> K1(TemperatureGillooly)
    K1 --> S1[temperature<br/>transformed]

    %% 2. GlobalMask
    S1 --> K2(GlobalMask)
    K2 --> S2[global_mask]

    %% 3. MaskByFunctionalGroup
    S2 --> K3(MaskByFunctionalGroup)
    P6 --> K3
    P7 --> K3
    K3 --> S3[mask_by_fgroup]

    %% 4. DayLength
    K4(DayLength)
    P10 --> K4
    K4 --> S4[day_length]

    %% 5. AverageTemperature
    S1 --> K5(AverageTemperature)
    S4 --> K5
    S3 --> K5
    P6 --> K5
    P7 --> K5
    K5 --> S5[avg_temperature_by_fgroup]

    %% 6. PrimaryProductionByFgroup
    F2 --> K6(PrimaryProductionByFgroup)
    P1 --> K6
    K6 --> S6[primary_production_by_fgroup]

    %% 7. MinTemperatureByCohort
    K7(MinTemperatureByCohort)
    P9 --> K7
    P4 --> K7
    P5 --> K7
    K7 --> S7[min_temperature]

    %% 8. MaskTemperature
    S5 --> K8(MaskTemperature)
    S7 --> K8
    K8 --> S8[mask_temperature]

    %% 9. MortalityField
    S5 --> K9(MortalityField)
    P2 --> K9
    P3 --> K9
    K9 --> S9[mortality_field]

    %% ===== NUMBA COMPILED KERNELS =====
    subgraph NUMBA[" "]
        direction TB

        %% 10. Production
        K10(Production)
        K10 --> S10[recruited]

        %% 11. Biomass
        S10 --> K11(Biomass)
        K11 --> S11[biomass]
    end

    S6 --> K10
    S8 --> K10
    P8 --> K10
    F3 -.-> K10

    S9 --> K11
    P13 --> K11
    F4 -.-> K11

    %% ===== OUTPUT =====
    S11 --> OUTPUT[[OUTPUT<br/>biomass]]

    %% ===== STYLING =====
    classDef forcingStyle fill:#e1f5ff,stroke:#0288d1,stroke-width:2px
    classDef paramStyle fill:#fff3e0,stroke:#f57c00,stroke-width:2px
    classDef kernelStyle fill:#f3e5f5,stroke:#7b1fa2,stroke-width:2px
    classDef stateStyle fill:#e8f5e9,stroke:#388e3c,stroke-width:2px
    classDef numbaStyle fill:#ffebee,stroke:#c62828,stroke-width:3px
    classDef outputStyle fill:#fff9c4,stroke:#f9a825,stroke-width:3px

    class F1,F2,F3,F4 forcingStyle
    class P1,P2,P3,P4,P5,P6,P7,P8,P9,P10,P13 paramStyle
    class K1,K2,K3,K4,K5,K6,K7,K8,K9,K10,K11 kernelStyle
    class S1,S2,S3,S4,S5,S6,S7,S8,S9,S10,S11 stateStyle
    class NUMBA numbaStyle
    class OUTPUT outputStyle
```

## Légende

### Éléments du diagramme

- **Forcings (bleu clair)** : Données d'entrée du modèle
  - `temperature` : Température par couche
  - `primary_production` : Production primaire
  - `initial_condition_production` : Conditions initiales de production (optionnel)
  - `initial_condition_biomass` : Conditions initiales de biomasse (optionnel)

- **Parameters (orange)** : Paramètres de configuration du modèle
  - Functional Group: `energy_transfert`, `lambda_temperature_0`, `gamma_lambda_temperature`, `tr_0`, `gamma_tr`, `day_layer`, `night_layer`, `timesteps_number`, `mean_timestep`
  - Kernel: `angle_horizon_sun`
  - Forcing: `timestep`

- **Kernel Units (violet)** : Fonctions de transformation sans gestion interne du temps
  - Transformation, masques, calculs de champs intermédiaires

- **Numba Kernels (rouge - subgraph)** : Fonctions avec JIT compilation et boucle FOR interne sur les pas de temps
  - `ProductionKernel` : Calcul du recrutement avec gestion des cohortes (expand_dims, ageing, production)
  - `BiomassKernel` : Intégration de la biomasse (biomass_euler_implicite)

- **States (vert)** : États intermédiaires produits par les kernels
  - Produits par chaque kernel et utilisés comme entrées par les suivants

- **Output (jaune)** : Résultat final du modèle
  - `biomass` : Biomasse finale par groupe fonctionnel

### Notes importantes

1. **Gestion du temps** : Les deux kernels dans le subgraph rouge (ProductionKernel et BiomassKernel) utilisent Numba pour la compilation JIT et contiennent des boucles FOR explicites sur les pas de temps.

2. **Conditions initiales optionnelles** : Les liens en pointillés (-.-> ) représentent les conditions initiales optionnelles qui ne sont utilisées que si elles sont fournies dans la configuration.

3. **Ordre d'exécution** : Le flux se lit de gauche à droite. Chaque kernel attend que tous ses inputs (forcings, parameters, states) soient disponibles avant de s'exécuter.

4. **Dépendances** : Les flèches montrent clairement les dépendances entre les différentes étapes du calcul.
