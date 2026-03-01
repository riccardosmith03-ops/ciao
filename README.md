[tool.json](https://github.com/user-attachments/files/25649817/tool.json)
{
  "name": "tool",
  "nodes": [
    {
      "parameters": {
        "httpMethod": "POST",
        "path": "pre-beta-milano",
        "responseMode": "lastNode",
        "options": {}
      },
      "type": "n8n-nodes-base.webhook",
      "typeVersion": 2.1,
      "position": [
        0,
        0
      ],
      "id": "01cc8819-136e-44f3-9d4a-f8a3c38c95ef",
      "name": "Webhook",
      "webhookId": "220fda28-17bb-45ce-9385-30aa7747c5b5"
    },
    {
      "parameters": {
        "jsCode": "// n8n Code node (JavaScript)\n// Prebeta Milano - Runtime Engine v1 (stateless compute from input)\n// Output: 1 item with json = report_output_schema_v1 (required + optional)\n\nfunction num(v) {\n  if (v === null || v === undefined || v === \"\") return null;\n  const n = typeof v === \"number\" ? v : Number(String(v).replace(\",\", \".\"));\n  return Number.isFinite(n) ? n : null;\n}\n\nfunction bool(v) {\n  if (typeof v === \"boolean\") return v;\n  if (v === null || v === undefined || v === \"\") return null;\n  const s = String(v).trim().toLowerCase();\n  if ([\"true\", \"1\", \"yes\", \"y\", \"si\", \"sì\"].includes(s)) return true;\n  if ([\"false\", \"0\", \"no\", \"n\"].includes(s)) return false;\n  return null;\n}\n\nfunction clamp(x, a, b) {\n  return Math.max(a, Math.min(b, x));\n}\n\nfunction eur(n) {\n  if (n === null || n === undefined || !Number.isFinite(n)) return \"—\";\n  const rounded = Math.round(n);\n  return \"€\" + String(rounded).replace(/\\B(?=(\\d{3})+(?!\\d))/g, \".\");\n}\n\nfunction pct(x, digits = 1) {\n  if (x === null || x === undefined || !Number.isFinite(x)) return \"—\";\n  return (x * 100).toFixed(digits) + \"%\";\n}\n\nfunction pctFromPercentNumber(x, digits = 1) {\n  if (x === null || x === undefined || !Number.isFinite(x)) return \"—\";\n  return x.toFixed(digits) + \"%\";\n}\n\nfunction safeStr(v) {\n  if (v === null || v === undefined) return \"\";\n  return String(v).trim();\n}\n\nfunction isOneOf(v, arr) {\n  return arr.includes(v);\n}\n\n// ✅ NEW: range helpers (ADD-ONLY)\nfunction roundTo(n, step) {\n  if (n === null || n === undefined || !Number.isFinite(n)) return null;\n  return Math.round(n / step) * step;\n}\nfunction eurRange(lo, hi, step = 10000) {\n  if (!Number.isFinite(lo) || !Number.isFinite(hi)) return \"—\";\n  const a = roundTo(lo, step);\n  const b = roundTo(hi, step);\n  return `${eur(a)} – ${eur(b)}`;\n}\n\n// --- CONTRACT DEFAULTS (fallback) ---\nconst ZONE_PRICE_OVERRIDES_EUR_PER_SQM = {\n  Duomo: 7000,\n  Brera: 6500,\n  Garibaldi: 5200,\n  Bovisa: 3500,\n  \"Quarto Oggiaro\": 2500,\n  \"Altro Milano\": 4200,\n};\n\nconst RENOVATION_COST_PER_SQM = {\n  none: 0, // ✅ NEW\n  light: 600,\n  medium: 900,\n  heavy: 1200,\n};\n\nconst STANDARD_MONTHLY_COSTS = {\n  pulizie: 100,\n  check_in: 50,\n  bollette: 200,\n};\n\nconst DEFAULT_PLATFORM_FEE_PERCENT = 15;\n\nconst DEFAULT_OCCUPANCY_BY_ZONE = {\n  Duomo: 80,\n  Brera: 78,\n  Garibaldi: 75,\n  Bovisa: 68,\n  \"Quarto Oggiaro\": 55,\n  \"Altro Milano\": 70,\n};\n\nconst RENT_BASELINES = {\n  Duomo: { long: 1900, short: 2800 },\n  Brera: { long: 1700, short: 2400 },\n  Garibaldi: { long: 1300, short: 1800 },\n  Bovisa: { long: 900, short: 1200 },\n  \"Quarto Oggiaro\": { long: 750, short: 900 },\n  \"Altro Milano\": { long: 1100, short: 1500 },\n};\n\nconst DECISION_RULES = {\n  opportunity_high_confidence_threshold: 0.10, // 10%\n  stress_factor_score: 0.8,\n  fallback_purchase_costs_percent_of_price: 0.07,\n  if_no_comparables_max_confidence: 0.55,\n  if_zone_is_altro_milano_max_confidence: 0.65,\n\n  // ✅ NEW: soglia conf più bassa se comparabili utente sono utilizzabili\n  opportunity_min_confidence_with_user_comps: 0.45,\n  // ✅ NEW: se spread è enorme possiamo accettare con conf più bassa\n  opportunity_very_strong_threshold: 0.25, // 25%\n\n  // ✅ NEW: se è troppo caro, NON è VERIFY: è NO_GO\n  overpriced_reject_threshold: -0.15, // -15% vs mid => NO_GO\n  overpriced_red_threshold: -0.25,    // -25% vs mid => RED + NO_GO\n};\n\n// --- INPUT ---\nif (!items || items.length === 0) {\n  return [{\n    json: {\n      error: \"NO_INPUT_ITEMS\",\n      hint: \"Esegui il nodo precedente (Set/Webhook) oppure usa 'Pin data' sul nodo precedente.\"\n    }\n  }];\n}\n\nconst input = items[0].json || {};\n\n// --- BENCHMARK READ ---\nconst benchmark = input.benchmark_milano_v1 || null;\nconst benchmark_present = !!benchmark;\n\n// --- BENCH TABLES (fallback-safe) ---\nconst ZONE_PRICE_OVERRIDES_EUR_PER_SQM_BENCH =\n  (benchmark && benchmark.zone_price_overrides_eur_per_sqm_mid_for_code)\n    ? benchmark.zone_price_overrides_eur_per_sqm_mid_for_code\n    : ZONE_PRICE_OVERRIDES_EUR_PER_SQM;\n\nconst LOW_HIGH_BAND_PCT_BENCH =\n  (benchmark && Number.isFinite(Number(benchmark.low_high_band_pct)))\n    ? Number(benchmark.low_high_band_pct)\n    : 0.07;\n\nconst MACRO_MARKET_PPSQM_MID_BENCH =\n  (benchmark && benchmark.macro_market && Number.isFinite(Number(benchmark.macro_market.sale_price_eur_per_sqm_mid)))\n    ? Number(benchmark.macro_market.sale_price_eur_per_sqm_mid)\n    : (ZONE_PRICE_OVERRIDES_EUR_PER_SQM[\"Altro Milano\"] ?? 4200);\n\nconst RENOVATION_COST_PER_SQM_BENCH =\n  (benchmark && benchmark.renovation_costs_eur_per_sqm_by_level)\n    ? benchmark.renovation_costs_eur_per_sqm_by_level\n    : RENOVATION_COST_PER_SQM;\n\nconst STANDARD_MONTHLY_COSTS_BENCH =\n  (benchmark && benchmark.short_term && benchmark.short_term.standard_monthly_costs_eur)\n    ? benchmark.short_term.standard_monthly_costs_eur\n    : STANDARD_MONTHLY_COSTS;\n\nconst DEFAULT_PLATFORM_FEE_PERCENT_BENCH =\n  (benchmark && benchmark.short_term && Number.isFinite(Number(benchmark.short_term.default_platform_fee_percent)))\n    ? Number(benchmark.short_term.default_platform_fee_percent)\n    : DEFAULT_PLATFORM_FEE_PERCENT;\n\nconst DEFAULT_OCCUPANCY_BY_ZONE_BENCH =\n  (benchmark && benchmark.short_term && benchmark.short_term.occupancy_by_zone_percent)\n    ? benchmark.short_term.occupancy_by_zone_percent\n    : DEFAULT_OCCUPANCY_BY_ZONE;\n\nconst RENT_BASELINES_BENCH =\n  (benchmark && benchmark.rent_baselines_by_zone_monthly_eur)\n    ? benchmark.rent_baselines_by_zone_monthly_eur\n    : RENT_BASELINES;\n\n// --- ZONE NORMALIZATION (FIX CRITICO) ---\nfunction normalizeZoneKey(z) {\n  const s = safeStr(z).toLowerCase().trim();\n  const map = {\n    \"duomo\": \"Duomo\",\n    \"brera\": \"Brera\",\n    \"porta venezia\": \"Porta Venezia\",\n    \"portavenezia\": \"Porta Venezia\",\n    \"garibaldi\": \"Garibaldi\",\n    \"isola\": \"Isola\",\n    \"navigli\": \"Navigli\",\n    \"bovisa\": \"Bovisa\",\n    \"quarto oggiaro\": \"Quarto Oggiaro\",\n    \"altro milano\": \"Altro Milano\",\n    \"altro\": \"Altro Milano\",\n  };\n  return map[s] || safeStr(z); // se già arriva \"Duomo\" torna \"Duomo\"\n}\n\n// Normalize expected fields from your user_input_schema_v1\nconst address_text = safeStr(input.address_text || input.address || input.indirizzo);\nconst zone_tag_raw = safeStr(input.zone_tag);\nconst zone_key_used = normalizeZoneKey(zone_tag_raw);\n\nconst property_type = safeStr(input.property_type || \"\");\nconst surface_sqm = num(input.surface_sqm);\nconst floor = num(input.floor);\nconst elevator = bool(input.elevator);\nconst condition = safeStr(input.condition);\nconst exposure = safeStr(input.exposure || \"mixed\");\nconst asking_price_eur = num(input.asking_price_eur);\nconst expected_renovation_level = safeStr(input.expected_renovation_level);\nconst renovation_budget_override_eur = num(input.renovation_budget_override_eur);\nconst purchase_costs_override_eur = num(input.purchase_costs_override_eur);\nconst condominium_monthly_eur = num(input.condominium_monthly_eur);\n\nconst rent_long_term_gross_eur_in = num(input.rent_long_term_gross_eur);\nconst rent_short_term_gross_eur_in = num(input.rent_short_term_gross_eur);\nconst short_term_expected_occupancy = num(input.short_term_expected_occupancy);\nconst short_term_platform_fees_percent = num(input.short_term_platform_fees_percent);\n\nconst stress_tolerance_0_1 = num(input.stress_tolerance_0_1);\nconst owner_responsiveness_0_1 = num(input.owner_responsiveness_0_1);\nconst timeline_urgency = safeStr(input.timeline_urgency || \"medium\");\n\nconst user_email = safeStr(input.email || input.user_email || input.mail || \"\");\n\n// ✅ NEW: renovation level canonical + defaults by condition\nlet expected_reno_level = expected_renovation_level;\nif (condition === \"renovated\") expected_reno_level = \"none\";\nif (condition === \"habitable\" && !expected_reno_level) expected_reno_level = \"light\";\nif (condition === \"to_renovate\" && !expected_reno_level) expected_reno_level = \"medium\";\n\n// --- QUALITY GATES (R0_VALIDATE) ---\nconst validation_errors = [];\nconst warnings = [];\n\nif (!benchmark_present) warnings.push(\"⚠️ benchmark_milano_v1 non presente: sto usando fallback hardcoded (controlla Merge).\");\n\nif (!address_text || address_text.length < 3) validation_errors.push(\"Indirizzo mancante o troppo corto.\");\nif (!zone_tag_raw) validation_errors.push(\"Zona (zone_tag) mancante.\");\nif (!Number.isFinite(surface_sqm) || surface_sqm < 10 || surface_sqm > 400) validation_errors.push(\"Mq (surface_sqm) fuori range (10-400).\");\nif (!condition) validation_errors.push(\"Stato immobile (condition) mancante.\");\nif (!Number.isFinite(asking_price_eur) || asking_price_eur < 10000 || asking_price_eur > 10000000) validation_errors.push(\"Prezzo richiesto (asking_price_eur) fuori range.\");\nif (!Number.isFinite(stress_tolerance_0_1) || stress_tolerance_0_1 < 0 || stress_tolerance_0_1 > 1) validation_errors.push(\"Tolleranza stress (0-1) mancante o fuori range.\");\nif (!expected_reno_level || !isOneOf(expected_reno_level, [\"none\", \"light\", \"medium\", \"heavy\"])) { // ✅ CHANGED\n  validation_errors.push(\"Livello ristrutturazione (expected_renovation_level) mancante o non valido (none/light/medium/heavy).\");\n}\n\nconst validation_status = validation_errors.length > 0 ? \"blocked\" : \"ok\";\n\nif (validation_status === \"blocked\") {\n  const report_email_text = [\n    \"REPORT PREBETA (BLOCCATO)\",\n    \"\",\n    `Immobile: ${address_text || \"—\"} (${zone_tag_raw || \"—\"}) - ${surface_sqm ?? \"—\"} mq`,\n    `Prezzo richiesto: ${eur(asking_price_eur)}`,\n    \"\",\n    \"Motivo blocco:\",\n    ...validation_errors.map(e => `- ${e}`),\n    \"\",\n    \"Note: Compila i campi mancanti e riprova.\"\n  ].join(\"\\n\");\n\n  return [{\n    json: {\n      deal_classification: \"red\",\n      final_decision_flag: \"NO_GO\",\n      decision_reasoning_text: \"Report bloccato: dati critici mancanti o fuori range.\",\n      market_value_low_eur: null,\n      market_value_mid_eur: null,\n      market_value_high_eur: null,\n      market_value_confidence_score: 0.0,\n      asking_price_eur,\n      spread_vs_market_mid_eur: null,\n      spread_vs_market_mid_percent: null,\n      capital_invested_estimated_eur: null,\n      renovation_estimated_eur: null,\n      rent_long_term_estimated_gross_eur: null,\n      rent_short_term_estimated_gross_eur: null,\n      rent_short_term_estimated_net_eur: null,\n      roi_long_term_estimated_percent: null,\n      roi_short_term_estimated_percent: null,\n      recommended_rent_type: null,\n      stress_score_used: null,\n      report_email_text,\n      warnings,\n      validation_errors,\n      missing_data_list: validation_errors,\n      user_email,\n\n      // extra pass-through (consistenza output)\n      occupancy_used_percent: null,\n      platform_fee_used_percent: null,\n      expected_monthly_net_eur: null,\n      condominium_monthly_eur,\n      property_type,\n      exposure,\n      timeline_urgency,\n      benchmark_present,\n      benchmark_zone_hit: null,\n      benchmark_used_zone_ppsqm: null,\n      benchmark_low_high_band_pct_used: LOW_HIGH_BAND_PCT_BENCH,\n      comparables_summary: input.comparables_summary || null,\n\n      input_echo: {\n        address_text,\n        zone_tag: zone_tag_raw,\n        zone_key_used,\n        property_type,\n        surface_sqm,\n        floor,\n        elevator,\n        condition,\n        asking_price_eur,\n        expected_renovation_level: expected_reno_level, // ✅ CHANGED\n        stress_tolerance_0_1,\n        owner_responsiveness_0_1,\n        timeline_urgency,\n        user_email,\n      },\n    }\n  }];\n}\n\n// --- R1_CORE_FILTERS ---\nconst price_per_sqm_eur = asking_price_eur / surface_sqm;\n\nconst small_unit = surface_sqm <= 60;\nif (!small_unit) warnings.push(\"Mq > 60: liquidità potenzialmente più bassa (non è blocco).\");\n\n// sanity check prezzo/mq vs benchmark zona (NOW WITH NORMALIZED KEY)\nconst zone_benchmark_ppsqm =\n  ZONE_PRICE_OVERRIDES_EUR_PER_SQM_BENCH[zone_key_used] ??\n  ZONE_PRICE_OVERRIDES_EUR_PER_SQM_BENCH[\"Altro Milano\"] ??\n  MACRO_MARKET_PPSQM_MID_BENCH;\n\nconst benchmark_zone_hit = Object.prototype.hasOwnProperty.call(ZONE_PRICE_OVERRIDES_EUR_PER_SQM_BENCH, zone_key_used);\n\nif (Number.isFinite(zone_benchmark_ppsqm)) {\n  const ratio = price_per_sqm_eur / zone_benchmark_ppsqm;\n  if (ratio > 1.4) warnings.push(`Prezzo/mq molto alto vs benchmark zona (${Math.round(zone_benchmark_ppsqm)} €/mq). Potrebbe essere caro o avere caratteristiche premium non inserite.`);\n  if (ratio < 0.6) warnings.push(`Prezzo/mq molto basso vs benchmark zona (${Math.round(zone_benchmark_ppsqm)} €/mq). Possibile opportunità o problema nascosto: verifica.`);\n}\n\nlet deal_classification = \"green\";\nlet early_reject_flag = false;\n\nconst ratio2 = price_per_sqm_eur / zone_benchmark_ppsqm;\n\n// ✅ OPTION B: soglia più alta per renovated, senza bypass totale (CHANGE MINIMA)\nconst maxRatio = (condition === \"renovated\") ? 2.0 : 1.8;\nif (ratio2 > maxRatio) {\n  deal_classification = \"red\";\n  early_reject_flag = true;\n}\n\n// --- MICRO PATCH: comparables_summary (dal merge) ---\nconst comps = input.comparables_summary || null;\nconst comps_count = Number(comps?.count ?? 0);\nconst comps_median_ppsqm = Number(comps?.median_ppsqm ?? NaN);\n\n// --- R2_MARKET_VALUE ---\nfunction conditionAdjPercent(cond) {\n  if (cond === \"renovated\") return +0.08;\n  if (cond === \"to_renovate\") return -0.08;\n  return 0;\n}\nfunction elevatorAdjPercent(hasElevator) {\n  if (hasElevator === false) return -0.05;\n  return 0;\n}\nfunction floorAdjPercent(f) {\n  if (!Number.isFinite(f)) return 0;\n  if (f <= 0) return -0.03;\n  if (f >= 5) return +0.02;\n  return 0;\n}\n\nlet base_ppsqm = zone_benchmark_ppsqm;\n\nconst comps_is_usable =\n  comps_count >= 2 &&\n  Number.isFinite(comps_median_ppsqm) &&\n  comps_median_ppsqm > 2000 &&\n  comps_median_ppsqm < 25000;\n\nif (comps_is_usable) {\n  // blend semplice: 70% benchmark zona + 30% comparabili\n  base_ppsqm = (0.70 * zone_benchmark_ppsqm) + (0.30 * comps_median_ppsqm);\n}\n\nconst adj =\n  conditionAdjPercent(condition) +\n  elevatorAdjPercent(elevator) +\n  floorAdjPercent(floor);\n\nconst adjusted_ppsqm = base_ppsqm * (1 + adj);\n\nconst market_value_mid_eur = adjusted_ppsqm * surface_sqm;\nconst market_value_low_eur = market_value_mid_eur * (1 - LOW_HIGH_BAND_PCT_BENCH);\nconst market_value_high_eur = market_value_mid_eur * (1 + LOW_HIGH_BAND_PCT_BENCH);\n\n// ✅ NEW: pretty range text for market value (ADD-ONLY)\nconst market_value_range_text = eurRange(market_value_low_eur, market_value_high_eur, 10000);\n\nlet market_value_confidence_score = 0.35;\nlet market_value_method_used = \"public_zone_benchmark\";\nlet market_value_notes_short = \"Stima prebeta basata su benchmark zona + aggiustamenti (stato/ascensore/piano). Confidenza bassa senza comparabili.\";\n\nmarket_value_confidence_score = Math.min(market_value_confidence_score, DECISION_RULES.if_no_comparables_max_confidence);\nif (zone_key_used === \"Altro Milano\") {\n  market_value_confidence_score = Math.min(market_value_confidence_score, DECISION_RULES.if_zone_is_altro_milano_max_confidence);\n  market_value_notes_short += \" Zona 'Altro Milano': benchmark più grezzo.\";\n}\nmarket_value_confidence_score = clamp(market_value_confidence_score, 0.2, 0.9);\n\nif (comps_is_usable) {\n  const boost = Number.isFinite(Number(comps?.confidence_boost)) ? Number(comps.confidence_boost) : 0.1;\n  market_value_confidence_score = clamp(market_value_confidence_score + boost, 0.2, 0.9);\n  market_value_method_used = \"zone_benchmark_blend_with_user_comparables\";\n  market_value_notes_short = \"Stima prebeta basata su benchmark zona + comparabili utente (blend) + aggiustamenti (stato/ascensore/piano).\";\n}\n\n// --- R3_CAPITAL_INVESTED ---\nconst purchase_costs_estimated_eur =\n  purchase_costs_override_eur !== null\n    ? purchase_costs_override_eur\n    : asking_price_eur * DECISION_RULES.fallback_purchase_costs_percent_of_price;\n\nconst renovation_estimated_eur =\n  renovation_budget_override_eur !== null\n    ? renovation_budget_override_eur\n    : surface_sqm * (RENOVATION_COST_PER_SQM_BENCH[expected_reno_level] ?? 900); // ✅ CHANGED\n\n// ✅ NEW: simple renovation \"da-a\" (ADD-ONLY, no logic change)\nconst renovation_low_eur = renovation_estimated_eur * 0.90;\nconst renovation_high_eur = renovation_estimated_eur * 1.10;\nconst renovation_range_text = eurRange(renovation_low_eur, renovation_high_eur, 1000);\n\nconst capital_invested_estimated_eur = asking_price_eur + purchase_costs_estimated_eur + renovation_estimated_eur;\n\n// --- R4_OPPORTUNITY ---\nconst spread_vs_market_mid_eur = market_value_mid_eur - asking_price_eur;\nconst spread_vs_market_mid_percent = (market_value_mid_eur - asking_price_eur) / market_value_mid_eur;\n\n// ✅ NEW: overpriced guardrail (ADD-ONLY)\nconst overpriced = Number.isFinite(spread_vs_market_mid_percent) &&\n  spread_vs_market_mid_percent <= DECISION_RULES.overpriced_reject_threshold;\n\nconst overpriced_red = Number.isFinite(spread_vs_market_mid_percent) &&\n  spread_vs_market_mid_percent <= DECISION_RULES.overpriced_red_threshold;\n\nif (overpriced_red) {\n  deal_classification = \"red\";\n  early_reject_flag = true;\n}\n\n// --- R5_RENT_ESTIMATES ---\nconst zoneRentBaseline = RENT_BASELINES_BENCH[zone_key_used] ?? RENT_BASELINES_BENCH[\"Altro Milano\"] ?? RENT_BASELINES[\"Altro Milano\"];\n\nlet rent_long_term_estimated_gross_eur =\n  rent_long_term_gross_eur_in !== null ? rent_long_term_gross_eur_in : zoneRentBaseline.long;\n\nlet rent_short_term_estimated_gross_eur =\n  rent_short_term_gross_eur_in !== null ? rent_short_term_gross_eur_in : zoneRentBaseline.short;\n\nconst occupancy_used_percent =\n  short_term_expected_occupancy !== null\n    ? short_term_expected_occupancy\n    : (DEFAULT_OCCUPANCY_BY_ZONE_BENCH[zone_key_used] ?? DEFAULT_OCCUPANCY_BY_ZONE_BENCH[\"Altro Milano\"] ?? DEFAULT_OCCUPANCY_BY_ZONE[\"Altro Milano\"]);\n\nconst platform_fee_used_percent =\n  short_term_platform_fees_percent !== null\n    ? short_term_platform_fees_percent\n    : DEFAULT_PLATFORM_FEE_PERCENT_BENCH;\n\nconst standard_costs_total =\n  (STANDARD_MONTHLY_COSTS_BENCH.pulizie || 0) +\n  (STANDARD_MONTHLY_COSTS_BENCH.check_in || 0) +\n  (STANDARD_MONTHLY_COSTS_BENCH.bollette || 0);\n\n// --- R7_STRESS_AND_RENT_DECISION ---\nconst estimated_stress_score = 1 - stress_tolerance_0_1;\n\nconst short_term_after_platform_fee =\n  rent_short_term_estimated_gross_eur * (1 - platform_fee_used_percent / 100);\n\nlet short_term_base_net_eur = short_term_after_platform_fee - standard_costs_total;\n\nconst stress_penalty_eur =\n  estimated_stress_score > DECISION_RULES.stress_factor_score\n    ? 0.15 * rent_short_term_estimated_gross_eur\n    : 0;\n\nlet rent_short_term_estimated_net_eur = short_term_base_net_eur - stress_penalty_eur;\n\nlet recommended_rent_type = null;\nlet expected_monthly_net_eur = null;\n\nif (Number.isFinite(rent_long_term_estimated_gross_eur) && Number.isFinite(rent_short_term_estimated_net_eur)) {\n  recommended_rent_type =\n    rent_short_term_estimated_net_eur <= rent_long_term_estimated_gross_eur\n      ? \"affitto_lungo\"\n      : \"affitto_breve\";\n\n  expected_monthly_net_eur =\n    recommended_rent_type === \"affitto_lungo\"\n      ? rent_long_term_estimated_gross_eur\n      : rent_short_term_estimated_net_eur;\n} else {\n  recommended_rent_type = \"unknown\";\n}\n\n// --- R6_ROI ---\nconst roi_long_term_estimated_percent =\n  rent_long_term_estimated_gross_eur > 0\n    ? (rent_long_term_estimated_gross_eur * 12) / capital_invested_estimated_eur * 100\n    : null;\n\nconst roi_short_term_estimated_percent =\n  rent_short_term_estimated_net_eur > 0\n    ? (rent_short_term_estimated_net_eur * 12) / capital_invested_estimated_eur * 100\n    : null;\n\nconst payback_years_long_term_estimated =\n  rent_long_term_estimated_gross_eur > 0 ? capital_invested_estimated_eur / (rent_long_term_estimated_gross_eur * 12) : null;\n\nconst payback_years_short_term_estimated =\n  rent_short_term_estimated_net_eur > 0 ? capital_invested_estimated_eur / (rent_short_term_estimated_net_eur * 12) : null;\n\n// --- R8_FINAL_DECISION (✅ FIX: overpriced => NO_GO) ---\nlet final_decision_flag = \"VERIFY\";\nlet decision_reasoning_text = \"\";\n\n// ✅ NEW: se è overpriced NON è VERIFY\nif (overpriced) {\n  final_decision_flag = \"NO_GO\";\n  decision_reasoning_text = `Sovrapprezzato vs stima mercato: ${pct(spread_vs_market_mid_percent, 1)} (mid).`;\n} else if (deal_classification === \"red\") {\n  final_decision_flag = \"NO_GO\";\n  decision_reasoning_text = \"Violazione regola core (prezzo/mq troppo alto vs benchmark senza condizioni premium).\";\n} else {\n  const opp = spread_vs_market_mid_percent;\n  const conf = market_value_confidence_score;\n\n  const conf_needed = comps_is_usable\n    ? DECISION_RULES.opportunity_min_confidence_with_user_comps\n    : 0.55;\n\n  const very_strong = opp >= DECISION_RULES.opportunity_very_strong_threshold;\n\n  if (opp >= DECISION_RULES.opportunity_high_confidence_threshold && conf >= conf_needed) {\n    final_decision_flag = \"GO\";\n    decision_reasoning_text = comps_is_usable\n      ? \"Sotto-prezzato vs stima (con comparabili utente) e confidenza sufficiente.\"\n      : \"Sotto-prezzato vs stima mercato e confidenza sufficiente.\";\n  } else if (very_strong && conf >= DECISION_RULES.opportunity_min_confidence_with_user_comps) {\n    final_decision_flag = \"GO\";\n    decision_reasoning_text = comps_is_usable\n      ? \"Spread molto forte vs stima (con comparabili) e conf accettabile: GO con due diligence.\"\n      : \"Spread molto forte vs stima ma conf limitata: GO solo con due diligence.\";\n  } else {\n    final_decision_flag = \"VERIFY\";\n    if (comps_is_usable) {\n      decision_reasoning_text = \"Serve verifica: confidenza ancora bassa/non abbastanza alta anche se i comparabili sono stati usati.\";\n    } else if (comps && comps_count > 0) {\n      decision_reasoning_text = \"Serve verifica: comparabili presenti ma non utilizzabili (confidenza bassa) o spread non abbastanza forte.\";\n    } else {\n      decision_reasoning_text = \"Serve verifica: mancano comparabili (confidenza bassa) o spread non abbastanza forte.\";\n    }\n  }\n}\n\ndecision_reasoning_text += ` ROI stimata: lungo ${pctFromPercentNumber(roi_long_term_estimated_percent)}, breve (netto) ${pctFromPercentNumber(roi_short_term_estimated_percent)}.`;\ndecision_reasoning_text += ` Stress stimato: ${pct(estimated_stress_score, 0)} (tolleranza ${pct(stress_tolerance_0_1, 0)}).`;\ndecision_reasoning_text += ` Strategia affitto consigliata: ${recommended_rent_type}.`;\n\nif (owner_responsiveness_0_1 !== null) {\n  decision_reasoning_text += ` Reattività proprietario: ${pct(owner_responsiveness_0_1, 0)}.`;\n}\n\n// --- R9_REPORT_RENDER ---\nconst missing_data_list = [];\nif (rent_long_term_gross_eur_in === null) missing_data_list.push(\"Affitto lungo inserito dall’utente (stimato con fallback).\");\nif (rent_short_term_gross_eur_in === null) missing_data_list.push(\"Affitto breve inserito dall’utente (stimato con fallback).\");\n\nif (!comps || comps_count <= 0) {\n  missing_data_list.push(\"Comparabili: non presenti (confidenza ridotta).\");\n} else if (!comps_is_usable) {\n  missing_data_list.push(\"Comparabili: presenti ma non utilizzabili (confidenza ridotta).\");\n} else {\n  missing_data_list.push(`Comparabili: utilizzati (n=${comps_count}, median=${Math.round(comps_median_ppsqm)} €/mq).`);\n}\n\nconst snapshot_line = `Immobile: ${address_text} (${zone_tag_raw}) - ${surface_sqm} mq - piano ${floor ?? \"—\"} - ascensore: ${elevator === null ? \"—\" : (elevator ? \"sì\" : \"no\")} - stato: ${condition}`;\n\nconst report_email_text = [\n  \"REPORT PREBETA – Milano (stima indicativa)\",\n  \"\",\n  snapshot_line,\n  \"\",\n  `Prezzo richiesto: ${eur(asking_price_eur)}  |  Prezzo/mq: ${eur(price_per_sqm_eur)}/mq`,\n  \"\",\n  \"1) Valore di mercato (stima)\",\n  `- Metodo: ${market_value_method_used}`,\n  // ✅ CHANGED: aggiunto \"da–a\" senza cambiare i calcoli\n  `- Range (da–a): ${market_value_range_text}  |  Mid: ${eur(market_value_mid_eur)}`,\n  `- Range (low/mid/high): ${eur(market_value_low_eur)} / ${eur(market_value_mid_eur)} / ${eur(market_value_high_eur)}`,\n  `- Confidenza: ${pct(market_value_confidence_score, 0)}`,\n  `- Note: ${market_value_notes_short}`,\n  \"\",\n  \"2) Opportunità prezzo\",\n  `- Scostamento vs mid: ${eur(spread_vs_market_mid_eur)} (${pct(spread_vs_market_mid_percent, 1)})`,\n  \"\",\n  \"3) Costi & capitale investito (stima)\",\n  // ✅ CHANGED: aggiunto range \"da–a\" ristrutturazione (±10%) senza cambiare logica\n  `- Ristrutturazione stimata (da–a): ${renovation_range_text}  |  Mid: ${eur(renovation_estimated_eur)} (${expected_reno_level})`,\n  `- Costi acquisto stimati: ${eur(purchase_costs_estimated_eur)}`,\n  `- Capitale investito stimato: ${eur(capital_invested_estimated_eur)}`,\n  \"\",\n  \"4) Rendite & ROI (stima)\",\n  `- Affitto lungo (lordo): ${eur(rent_long_term_estimated_gross_eur)} / mese  |  ROI lungo: ${pctFromPercentNumber(roi_long_term_estimated_percent)}`,\n  `- Affitto breve (lordo): ${eur(rent_short_term_estimated_gross_eur)} / mese`,\n  `  - Fee piattaforme: ${platform_fee_used_percent}% | Costi standard: ${eur(standard_costs_total)} | Penalty stress: ${eur(stress_penalty_eur)}`,\n  `  - Netto breve stimato: ${eur(rent_short_term_estimated_net_eur)} / mese  |  ROI breve (netto): ${pctFromPercentNumber(roi_short_term_estimated_percent)}`,\n  \"\",\n  \"5) Decisione\",\n  `- Classificazione: ${deal_classification.toUpperCase()}`,\n  `- Decisione: ${final_decision_flag}`,\n  `- Motivi: ${decision_reasoning_text}`,\n  \"\",\n  \"6) Dati mancanti / warning\",\n  ...(warnings.length ? warnings.map(w => `- ⚠️ ${w}`) : [\"- Nessun warning.\"]),\n  ...(missing_data_list.length ? missing_data_list.map(m => `- ℹ️ ${m}`) : []),\n  \"\",\n  \"Debug:\",\n  `- zone_tag_raw: ${zone_tag_raw}`,\n  `- zone_key_used: ${zone_key_used}`,\n  `- benchmark_zone_hit: ${benchmark_zone_hit ? \"yes\" : \"no\"}`,\n  `- zone_benchmark_ppsqm_used: ${Math.round(zone_benchmark_ppsqm)} €/mq`,\n  `- comparables_present: ${comps_count > 0 ? \"yes\" : \"no\"}`,\n  `- comps_is_usable: ${comps_is_usable ? \"yes\" : \"no\"}`,\n  `- comps_count: ${comps_count}`,\n  `- comps_median_ppsqm: ${Number.isFinite(comps_median_ppsqm) ? Math.round(comps_median_ppsqm) + \" €/mq\" : \"—\"}`,\n  `- base_ppsqm_used (pre-adj): ${Math.round(base_ppsqm)} €/mq`,\n  \"\",\n  \"Disclaimer: stime indicative, non è una perizia. Usa il report come supporto decisionale.\"\n].join(\"\\n\");\n\n// Optional HTML (basic)\nconst report_html = `\n<h2>Report Prebeta – Milano</h2>\n<p><strong>${snapshot_line}</strong></p>\n<h3>Valore di mercato</h3>\n<ul>\n  <li>Metodo: ${market_value_method_used}</li>\n  <li>Range (da–a): ${market_value_range_text} — Mid: ${eur(market_value_mid_eur)}</li>\n  <li>Range (low/mid/high): ${eur(market_value_low_eur)} / ${eur(market_value_mid_eur)} / ${eur(market_value_high_eur)}</li>\n  <li>Confidenza: ${pct(market_value_confidence_score, 0)}</li>\n  <li>Note: ${market_value_notes_short}</li>\n</ul>\n<h3>Opportunità</h3>\n<ul>\n  <li>Scostamento vs mid: ${eur(spread_vs_market_mid_eur)} (${pct(spread_vs_market_mid_percent, 1)})</li>\n</ul>\n<h3>Costi & capitale</h3>\n<ul>\n  <li>Ristrutturazione stimata (da–a): ${renovation_range_text} — Mid: ${eur(renovation_estimated_eur)} (${expected_reno_level})</li>\n  <li>Costi acquisto stimati: ${eur(purchase_costs_estimated_eur)}</li>\n  <li>Capitale investito stimato: ${eur(capital_invested_estimated_eur)}</li>\n</ul>\n<h3>Rendite & ROI</h3>\n<ul>\n  <li>Lungo (lordo): ${eur(rent_long_term_estimated_gross_eur)} / mese — ROI: ${pctFromPercentNumber(roi_long_term_estimated_percent)}</li>\n  <li>Breve (lordo): ${eur(rent_short_term_estimated_gross_eur)} / mese — Netto stimato: ${eur(rent_short_term_estimated_net_eur)} / mese — ROI: ${pctFromPercentNumber(roi_short_term_estimated_percent)}</li>\n  <li>Strategia consigliata: <strong>${recommended_rent_type}</strong></li>\n</ul>\n<h3>Decisione</h3>\n<ul>\n  <li>Classificazione: ${deal_classification.toUpperCase()}</li>\n  <li>Decisione: <strong>${final_decision_flag}</strong></li>\n  <li>Motivi: ${decision_reasoning_text}</li>\n</ul>\n<h3>Dati mancanti / warning</h3>\n<ul>\n  ${(warnings.length ? warnings.map(w => `<li>⚠️ ${w}</li>`).join(\"\") : \"<li>Nessun warning</li>\")}\n  ${(missing_data_list.length ? missing_data_list.map(m => `<li>ℹ️ ${m}</li>`).join(\"\") : \"\")}\n</ul>\n<p><em>Disclaimer: stime indicative, non è una perizia.</em></p>\n`;\n\n// --- FINAL OUTPUT ---\nreturn [{\n  json: {\n    deal_classification,\n    final_decision_flag,\n    decision_reasoning_text,\n\n    market_value_low_eur,\n    market_value_mid_eur,\n    market_value_high_eur,\n    market_value_confidence_score,\n\n    // ✅ NEW: output text range (ADD-ONLY)\n    market_value_range_text,\n\n    asking_price_eur,\n    spread_vs_market_mid_eur,\n    spread_vs_market_mid_percent,\n\n    capital_invested_estimated_eur,\n    renovation_estimated_eur,\n\n    // ✅ NEW: renovation range (ADD-ONLY)\n    renovation_low_eur,\n    renovation_high_eur,\n    renovation_range_text,\n\n    rent_long_term_estimated_gross_eur,\n    rent_short_term_estimated_gross_eur,\n    rent_short_term_estimated_net_eur,\n\n    roi_long_term_estimated_percent,\n    roi_short_term_estimated_percent,\n\n    recommended_rent_type,\n    stress_score_used: estimated_stress_score,\n\n    report_email_text,\n\n    payback_years_long_term_estimated,\n    payback_years_short_term_estimated,\n    market_value_method_used,\n    market_value_notes_short,\n    report_html,\n    missing_data_list,\n    warnings,\n\n    input_echo: {\n      address_text,\n      zone_tag: zone_tag_raw,\n      zone_key_used,\n      property_type,\n      surface_sqm,\n      floor,\n      elevator,\n      condition,\n      asking_price_eur,\n      expected_renovation_level: expected_reno_level, // ✅ CHANGED\n      stress_tolerance_0_1,\n      owner_responsiveness_0_1,\n      timeline_urgency,\n      user_email,\n    },\n\n    validation_errors,\n    validation_status,\n    user_email,\n    occupancy_used_percent,\n    platform_fee_used_percent,\n    expected_monthly_net_eur,\n    condominium_monthly_eur,\n    property_type,\n    exposure,\n    timeline_urgency,\n\n    benchmark_present,\n    benchmark_zone_hit,\n    benchmark_used_zone_ppsqm: zone_benchmark_ppsqm,\n    benchmark_low_high_band_pct_used: LOW_HIGH_BAND_PCT_BENCH,\n\n    // pass-through: così lo vedi anche nell'output JSON finale\n    comparables_summary: comps || null,\n\n    // extra debug output (consistenza / verificabilità)\n    comparables_present: comps_count > 0,\n    comps_is_usable,\n    comps_count,\n    comps_median_ppsqm,\n    base_ppsqm_used_pre_adj: base_ppsqm, // ✅ FIX: era variabile non definita\n\n    // ✅ NEW: extra debug (ADD-ONLY)\n    overpriced,\n    overpriced_red,\n  }\n}];\n"
      },
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        1408,
        0
      ],
      "id": "783d3894-0ecd-45fe-8c73-030c2b3603a1",
      "name": "Code in JavaScript1"
    },
    {
      "parameters": {
        "assignments": {
          "assignments": [
            {
              "id": "7a006224-37a8-49f5-af65-b8f15cc2b0d4",
              "name": "user_email",
              "value": "={{ ($json.user_email || \"\").toString().replace(/^=+/, \"\").trim() }}\n\n\n",
              "type": "string"
            },
            {
              "id": "bc673251-d9af-4402-bcff-3bda6dc14410",
              "name": "to_email",
              "value": "riccardosmith03@gmail.com",
              "type": "string"
            },
            {
              "id": "2e11d7a5-0029-4aaf-af77-e881461a96d2",
              "name": "email_subject",
              "value": "={{\n  \"Report pre-beta Milano - \" +\n  ($json.input_echo?.address_text || $json.input_echo?.zone_tag || \"Immobile\") +\n  ($json.input_echo?.property_type ? \" (\" + $json.input_echo.property_type + \")\" : \"\")\n}}\n\n",
              "type": "string"
            },
            {
              "id": "c653fac0-eb15-45aa-9fce-9acdc3769352",
              "name": "email_body",
              "value": "=={{$json.email_body}}\n",
              "type": "string"
            }
          ]
        },
        "options": {}
      },
      "type": "n8n-nodes-base.set",
      "typeVersion": 3.4,
      "position": [
        2032,
        0
      ],
      "id": "60e0bf44-09ce-4a9d-8b80-8695e9cacc72",
      "name": "Edit Fields1"
    },
    {
      "parameters": {
        "jsCode": "// n8n Code node (JavaScript) - Build Email Body (v2: uses input_echo first)\n\nfunction safe(v, fallback = \"N/D\") {\n  if (v === null || v === undefined) return fallback;\n  if (typeof v === \"function\") return fallback;\n  if (typeof v === \"string\" && v.trim() === \"\") return fallback;\n  return v;\n}\n\nfunction eur(v) {\n  const n = Number(v);\n  if (!Number.isFinite(n)) return \"N/D\";\n  return \"€\" + Math.round(n).toLocaleString(\"it-IT\");\n}\n\nfunction pct(v, decimals = 1) {\n  const n = Number(v);\n  if (!Number.isFinite(n)) return \"N/D\";\n  return n.toFixed(decimals) + \"%\";\n}\n\nfunction pctFromFraction(v, decimals = 1) {\n  const n = Number(v);\n  if (!Number.isFinite(n)) return \"N/D\";\n  return (n * 100).toFixed(decimals) + \"%\";\n}\n\nfunction yesNo(v) {\n  if (v === true) return \"sì\";\n  if (v === false) return \"no\";\n  return \"N/D\";\n}\n\nconst j = $json;\nconst ie = j.input_echo || {}; // <--- nuovo: input echo dal runtime engine\n\n// --- pick values (prefer input_echo, fallback to root)\nconst address = safe(ie.address_text ?? j.address_text, \"\");\nconst zone = safe(ie.zone_tag ?? j.zone_tag, \"N/D\");\nconst titlePlace = address ? address : (\"Zona \" + zone);\n\nconst sqm = ie.surface_sqm ?? j.surface_sqm;\nconst floor = ie.floor ?? j.floor;\nconst elevator = ie.elevator ?? j.elevator;\nconst condition = ie.condition ?? j.condition;\nconst propertyType = safe(ie.property_type ?? j.property_type, \"\");\n\n// market values (computed)\nconst mvLow = j.market_value_low_eur;\nconst mvMid = j.market_value_mid_eur;\nconst mvHigh = j.market_value_high_eur;\n\n// --- micro patch: confidenza in % (score 0-1)\nconst confPct = pctFromFraction(j.market_value_confidence_score, 0);\n\n// --- micro patch: scostamento anche in €\nconst spreadEur = eur(j.spread_vs_market_mid_eur);\n\nconst body = [\n  \"Analisi pre-beta immobile – Milano\",\n  \"\",\n  \"Immobile:\",\n  `${titlePlace}${propertyType ? \" – \" + propertyType : \"\"} – ${safe(sqm)} mq`,\n  `Piano ${safe(floor)} – Ascensore: ${yesNo(elevator)}`,\n  `Stato: ${safe(condition)}`,\n  \"\",\n  \"Valore di mercato stimato:\",\n  `Low: ${eur(mvLow)}`,\n  `Mid: ${eur(mvMid)}`,\n  `High: ${eur(mvHigh)}`,\n  `Confidenza: ${confPct}`,\n  \"\",\n  `Prezzo richiesto: ${eur(j.asking_price_eur)}`,\n  `Scostamento vs mercato (mid): ${spreadEur} (${pctFromFraction(j.spread_vs_market_mid_percent, 1)})`,\n  \"\",\n  `Capitale investito stimato: ${eur(j.capital_invested_estimated_eur)}`,\n  `Ristrutturazione stimata: ${eur(j.renovation_estimated_eur)}`,\n  \"\",\n  \"Rendimenti stimati:\",\n  `Affitto lungo ROI: ${pct(j.roi_long_term_estimated_percent, 1)}`,\n  `Affitto breve ROI: ${pct(j.roi_short_term_estimated_percent, 1)}`,\n  `Affitto consigliato: ${safe(j.recommended_rent_type)}`,\n  \"\",\n  `Stress operativo stimato: ${pctFromFraction(j.stress_score_used, 0)}`,\n  \"\",\n  `Decisione finale: ${safe(j.final_decision_flag)}`,\n  `Motivazione: ${safe(j.decision_reasoning_text, \"\")}`,\n  \"\",\n  \"Nota: report pre-beta, stime indicative, non è una perizia.\"\n].join(\"\\n\");\n\nreturn [{\n  json: {\n    ...j,\n    // manteniamo tutto + aggiungiamo/riscriviamo il body pulito\n    email_body: body\n  }\n}];\n\n\n\n"
      },
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        1616,
        0
      ],
      "id": "bd90d893-c2b4-4fc7-8a7d-32c2de6d3f1d",
      "name": "Code in JavaScript2"
    },
    {
      "parameters": {
        "sendTo": "={{$json.user_email.replace(/^=+/, '').trim()}}\n",
        "subject": "={{\"Report pre-beta Milano – \" + ($json.zone_tag || \"Immobile\")}}\n",
        "emailType": "text",
        "message": "={{$json.email_message.toString().replace(/^=+/, '').trim() }}\n\n",
        "options": {}
      },
      "type": "n8n-nodes-base.gmail",
      "typeVersion": 2.2,
      "position": [
        3344,
        0
      ],
      "id": "da539d24-2bf7-44c3-9040-0d6e2934e3b7",
      "name": "Send a message",
      "webhookId": "5c919542-bfe6-42ee-8685-2c619962a494",
      "credentials": {
        "gmailOAuth2": {
          "id": "iNtndvp66Ww2Cjvy",
          "name": "Gmail account"
        }
      }
    },
    {
      "parameters": {
        "modelId": {
          "__rl": true,
          "value": "gpt-4o-mini",
          "mode": "list",
          "cachedResultName": "GPT-4O-MINI"
        },
        "responses": {
          "values": [
            {
              "content": "=Rendi questo report un testo decisionale. Regole: non inventare nulla, copia i numeri identici, niente markdown, niente elenchi, non fare calcoli, non arrotondare, non usare mai la stima media e non usare mai “mid/media/medio”. Apri con questi campi SOLO elencati e senza spiegazioni: indirizzo (se c’è), zona, tipo immobile, mq, piano, ascensore, stato, prezzo richiesto. Poi descrivi in modo semplice (non tecnico) valore Da (valore possibile più basso) A (valore possibile più alto) usando SOLO i valori low e high presenti nel testo dando quindi il range del valore stimato, senza stima media, differenza percentuale rispetto al mercato, ritorno annuo stimato affitto a lungo termine, ritorno annuo stimato affitto a breve termine, confidenza, stress e costi di ristrutturazione. Trasforma tutto in un discorso umano finendo con un commento con linguaggio semplicissimo, non usare spread e roi come termini usa il loro significato, non citare la bassa confidenza, sii sicuro (non dire sono necessarie verifiche ma aggiungi informazioni per una maggior precisione).\nNon usare mai le parole: verificare, verifica, verifiche, approfondire, approfondimenti.Concludi SEMPRE con UNA SOLA riga finale, scegliendo ESATTAMENTE una di queste in base alla Decisione finale presente nel testo:\n- Se Decisione finale è GO: “Procedere = numeri ok e rischio sotto controllo”\n- Se Decisione finale è NO_GO: “Evitare = prezzo fuori mercato o ROI troppo basso”\n- Se Decisione finale è VERIFY: “Procedere = numeri interessanti, aggiungere dati per una stima più precisa”\ntesto:\n{{ $json.email_subject }}\n{{ $json.email_body }}\n"
            }
          ]
        },
        "builtInTools": {},
        "options": {}
      },
      "type": "@n8n/n8n-nodes-langchain.openAi",
      "typeVersion": 2.1,
      "position": [
        2480,
        0
      ],
      "id": "500de2a7-950c-470a-aaf3-26bb653b94b1",
      "name": "Message a model",
      "alwaysOutputData": true,
      "credentials": {
        "openAiApi": {
          "id": "NzZLAFPjpbpInFN9",
          "name": "OpenAi account"
        }
      }
    },
    {
      "parameters": {
        "jsCode": "function cleanEmail(v) {\n  return String(v || \"\")\n    .replace(/^=+/, \"\")\n    .replace(/\\r/g, \"\")\n    .trim();\n}\n\nlet human =\n  $json?.output?.[0]?.content?.[0]?.text ||\n  $json?.content?.[0]?.text ||\n  $json?.text ||\n  \"\";\n\n// FIX: trasforma \"\\n\" (testo) in newline reali\nhuman = human.replace(/\\\\n/g, \"\\n\").trim();\n\nconst edit = $('Edit Fields1').item.json;\n\nreturn [{\n  json: {\n    user_email: cleanEmail(edit.user_email),\n    email_subject: \"Report pre-beta Milano\",\n    email_message: human || \"Report non disponibile (OpenAI non ha restituito testo).\"\n  }\n}];\n\n\n"
      },
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        2976,
        -16
      ],
      "id": "c5034f9d-d7d1-482c-8c56-1181bb1b1c5d",
      "name": "Code in JavaScript3"
    },
    {
      "parameters": {
        "assignments": {
          "assignments": [
            {
              "id": "5dbc7658-633c-4550-989b-871d42b2db3d",
              "name": "benchmark_milano_v1",
              "value": "={   \"meta\": {     \"name\": \"benchmark_milano_v1\",     \"scope\": \"Milano only\",     \"years\": \"2025-2026\",     \"notes\": [       \"Dati riordinati e consolidati dal report condiviso.\",       \"Parte zona-specifica (vendita/affitto €/mq) include valori Gen 2026 (Wikicasa).\",       \"Altri blocchi (Airbnb, ristrutturazioni, costi acquisto, coefficienti) sono parametri 2025 riportati nel report.\"     ]   },   \"macro_market\": {     \"sale_price_milano_avg_eur_per_sqm_jan_2026\": 5615,     \"sale_price_milano_avg_range_2025_2026_eur_per_sqm\": \"5400-5600\",     \"sale_price_notes\": [       \"Zone centrali (Centro/CBD) oltre 10.000–11.000 €/mq; periferie molto più basse.\"     ],     \"starting_point_grid_patterns\": {       \"Centro_Quadrilatero_Duomo_Brera_eur_per_sqm\": \"9000-11000\",       \"PortaVenezia_CityLife_PortaNuova_eur_per_sqm\": \"7000-9000\",       \"Navigli_Bocconi_semicentro_eur_per_sqm\": \"6000-7000\",       \"Bovisa_QuartoOggiaro_periferie_eur_per_sqm\": \"2600-3200\"     },     \"trend_2025\": {       \"sale_growth_h1_2025_tecnocasa\": \"+1.4%\",       \"rents\": \"stabili/rallentamento\"     }   },   \"zones\": {     \"Duomo\": {       \"sale_mid_eur_per_sqm_jan_2026\": 10921,       \"rent_long_eur_per_sqm_month_jan_2026\": 30.97     },     \"Brera\": {       \"sale_mid_eur_per_sqm_jan_2026\": 10869,       \"rent_long_eur_per_sqm_month_jan_2026\": 27.42,       \"source_label_note\": \"Wikicasa: 'Cattolica, Sant Ambrogio, Magenta, Brera'\"     },     \"Porta Venezia\": {       \"sale_mid_eur_per_sqm_jan_2026\": 7895,       \"rent_long_eur_per_sqm_month_jan_2026\": 25.63,       \"source_label_note\": \"Wikicasa: 'Porta Venezia, Palestro, Monforte, Corridoni'\"     },     \"Città Studi\": {       \"sale_mid_eur_per_sqm_jan_2026\": 5776,       \"rent_long_eur_per_sqm_month_jan_2026\": 20.44     },     \"Garibaldi\": {       \"sale_mid_eur_per_sqm_jan_2026\": 9793,       \"rent_long_eur_per_sqm_month_jan_2026\": 28.85,       \"source_label_note\": \"Wikicasa: 'Porta Nuova, Moscova, Garibaldi…'\"     },     \"Isola\": {       \"sale_mid_eur_per_sqm_jan_2026\": 6073,       \"rent_long_eur_per_sqm_month_jan_2026\": 22.36,       \"source_label_note\": \"Wikicasa: 'Isola, Farini, Monumentale'\"     },     \"Navigli\": {       \"sale_mid_eur_per_sqm_jan_2026\": 6635,       \"rent_long_eur_per_sqm_month_jan_2026\": 21.82,       \"source_label_note\": \"Wikicasa: 'Porta Genova, Navigli, Darsena…'\"     },     \"Bovisa\": {       \"sale_mid_eur_per_sqm_jan_2026\": 4353,       \"rent_long_eur_per_sqm_month_jan_2026\": 19.68,       \"source_label_note\": \"Wikicasa: 'Bovisa, Dergano'\"     },     \"Quarto Oggiaro\": {       \"sale_mid_eur_per_sqm_jan_2026\": 2751,       \"rent_long_eur_per_sqm_month_jan_2026\": 18.83,       \"source_label_note\": \"Wikicasa: 'Quarto Oggiaro, Euromilano'\"     },     \"Altro Milano\": {       \"sale_mid_eur_per_sqm_jan_2026_fallback_city_avg\": 5885,       \"notes\": [         \"Wikicasa: media Milano Gen 2026 vs Gen 2025 = 5885 vs 5598 €/mq (+5%)\",         \"Nota extra riportata nel testo: vendita min 2668 €/mq e max 10205 €/mq (macro-city)\"       ]     }   },   \"zone_price_overrides_eur_per_sqm_mid_for_code\": {     \"Duomo\": 10921,     \"Brera\": 10869,     \"Porta Venezia\": 7895,     \"Città Studi\": 5776,     \"Garibaldi\": 9793,     \"Isola\": 6073,     \"Navigli\": 6635,     \"Bovisa\": 4353,     \"Quarto Oggiaro\": 2751,     \"Altro Milano\": 5885   },   \"range_rule_for_low_high\": {     \"low_multiplier\": 0.93,     \"high_multiplier\": 1.07,     \"note\": \"Regola già nel tuo motore: low=mid*0.93, high=mid*1.07 (±7%)\"   },   \"rents_long_macro_2025_2026\": {     \"milano_avg_eur_per_sqm_month_jan_2026\": \"22.32-23\",     \"centre_prime_eur_per_sqm_month\": \"30+\",     \"high_demand_eur_per_sqm_month\": \"23-25\",     \"benchmark_synthetic\": {       \"centro_eur_per_sqm_month\": \"~30\",       \"semicentro_navigli_porta_venezia_eur_per_sqm_month\": \"24-27\",       \"periferie_eur_per_sqm_month\": \"18-22\"     },     \"tendency\": [       \"2025 canoni stabili o leggermente in calo (stagionalità).\",       \"Gen 2026 lieve aumento YoY → domanda stabile o leggermente positiva nel 2026.\"     ]   },   \"rents_long_monthly_by_size_end_2025\": {     \"size_notes\": \"Mono ~30–40mq, Bilo ~50–60mq, Trilo ~80–90mq\",     \"Duomo/Brera\": {       \"mono_eur_month\": \"1100-1200\",       \"bilo_eur_month\": \"1600-1800\",       \"trilo_eur_month\": \"2400-2700\",       \"extra_note\": \"Centro ~31 €/mq (esempio 50mq ~1550€)\"     },     \"Garibaldi/Porta Nuova\": {       \"mono_eur_month\": \"~1000\",       \"bilo_eur_month\": \"1400-1500\",       \"trilo_eur_month\": \"2200-2300\"     },     \"Porta Venezia\": {       \"mono_eur_month\": \"~900\",       \"bilo_eur_month\": \"~1300\",       \"trilo_eur_month\": \"~1800\"     },     \"Navigli\": {       \"mono_eur_month\": \"~1000\",       \"bilo_eur_month\": \"~1400\",       \"trilo_eur_month\": \"~1850\"     },     \"Isola\": {       \"mono_eur_month\": \"900-1000\",       \"bilo_eur_month\": \"1300-1500\",       \"trilo_eur_month\": \"~2000\"     },     \"Bovisa\": {       \"mono_eur_month\": \"~700\",       \"bilo_eur_month\": \"~1000\",       \"trilo_eur_month\": \"~1300\"     },     \"Quarto Oggiaro\": {       \"mono_eur_month\": \"~600\",       \"bilo_eur_month\": \"~800\",       \"trilo_eur_month\": \"~1000\",       \"extra_note\": \"Stimato ~15% in meno rispetto a Bovisa (mono 700€)\"     }   },   \"rent_baselines_fill_method\": {     \"standard_sizes_sqm\": {       \"mono\": 40,       \"bilo\": 60,       \"trilo\": 80     },     \"formula\": \"rent_long_monthly = (rent_eur_per_sqm_month) * sqm\",     \"examples_mono_40sqm\": {       \"Duomo\": 1239,       \"Porta Venezia\": 1025,       \"Navigli\": 873     }   },   \"short_term_airbnb_2025\": {     \"zones_top\": {       \"Duomo/Brera\": {         \"occupancy_percent\": \"~80\",         \"adr_eur\": \"~180\",         \"adr_example_eur\": 186,         \"gross_monthly_potential_eur\": \"4000-5000\"       },       \"Garibaldi/Isola (Porta Nuova)\": {         \"occupancy_percent\": \"70-75\",         \"occupancy_example_percent\": 71,         \"adr_eur\": \"~120\",         \"adr_example_eur\": 115,         \"gross_monthly_potential_eur\": \"2500-3000\"       },       \"Porta Venezia\": {         \"occupancy_percent\": \"~70\",         \"adr_eur\": \"100-110\",         \"extra_note\": \"Nel testo: in linea con media città occ ~73%, ADR 130\"       },       \"Navigli\": {         \"occupancy_percent\": \"~75\",         \"adr_eur\": \"~120\",         \"adr_similar_zones_note\": \"ADR ~118\",         \"mono_gross_monthly_eur\": \"2000-2500\"       }     },     \"periphery_note\": \"Bovisa/Quarto Oggiaro: domanda turistica bassa → occupancy <50% e ADR <80; spesso sconsigliato.\",     \"platform_fee_percent\": 15,     \"monthly_variable_costs_estimates\": {       \"cleaning_cost_each_eur\": 18,       \"cleaning_guest_charge_example_eur\": 90,       \"bookings_per_month_example\": \"6-8\",       \"cleaning_monthly_mono_example_eur\": 150,       \"utilities_monthly_small_eur\": 100,       \"checkin_services_monthly_eur\": \"50-100\",       \"total_variable_monthly_mono_eur\": \"250-400\",       \"total_variable_monthly_bilo_eur\": \"400-600\"     }   },   \"renovation_costs_milano_2025_eur_per_sqm\": {     \"note_area_multiplier\": {       \"prestige_area\": \"+15%\",       \"periphery\": \"-10%\"     },     \"reno_light_eur_per_sqm\": \"300-500\",     \"reno_light_extra_note\": \"Nel testo appare anche fascia piccoli interventi 200-500 €/mq\",     \"reno_medium_eur_per_sqm\": \"600-900\",     \"reno_heavy_eur_per_sqm\": \"1000-1300\",     \"extras\": {       \"bathroom_eur\": \"7000-10000\",       \"bathroom_extra_note\": \"Citato anche: 7-9k opere, ~8000 in media\",       \"kitchen_eur\": \"5000-8000\",       \"kitchen_note\": \"Esclusi elettrodomestici di lusso\"     }   },   \"purchase_costs_percent_scenarios\": {     \"low_first_home\": {       \"percent_range\": \"6-8\",       \"components\": [         \"registro 2% su valore catastale (prima casa) (~3% del prezzo medio)\",         \"notaio ~1-1.5%\",         \"agenzia ~3%\"       ],       \"example\": \"300k → ~20k extra\",       \"note\": \"Acquisto da impresa: IVA 4% sostituisce registro\"     },     \"mid_second_home_typical\": {       \"percent_range\": \"10-12\",       \"components\": [         \"registro 9% oppure IVA 10% da costruttore\",         \"tasse ~6-9%\",         \"agenzia 2-3%\",         \"notaio 1-2%\"       ],       \"example\": \"300k → 30-35k extra\"     },     \"high_onerous\": {       \"percent_range\": \"14-16\",       \"components\": [         \"IVA 10% o 22% se lusso\",         \"provvigione ~3% + IVA\",         \"notaio ~1.5%\",         \"imposte fisse\"       ],       \"mortgage_addons\": [         \"+~2000€ notaio mutuo\",         \"imposta sostitutiva 0.25-2% del capitale mutuo\"       ]     },     \"summary_notes\": [       \"Prima casa da costruttore: IVA 4% + notaio + agenzia → ~8-9%\",       \"Seconda casa da privato: registro 9% + notaio + agenzia → ~12-13%\"     ]   },   \"adjustment_rules_percent\": {     \"condition\": {       \"to_renovate_vs_good\": \"-10%\",       \"recently_renovated_vs_good\": \"+5%\",       \"new_or_high_end_renovated_vs_good\": \"+10%\"     },     \"floor\": {       \"neutral_reference\": \"3rd floor with elevator = 0%\",       \"above_3rd\": \"+5% per floor (up to +10% at very high floors)\",       \"penthouse\": \"up to +20%\",       \"penalties\": {         \"2nd\": \"-3%\",         \"1st_or_ground\": \"-10%\",         \"basement_or_semibasement\": \"-25%\"       }     },     \"elevator\": {       \"missing\": \"-3% to -5% (especially above 2nd floor)\",       \"present\": \"+3% to +5%\"     },     \"exposure_luminosity\": {       \"very_bright_open_view\": \"+5% to +10%\",       \"partly_internal\": \"-5%\",       \"very_internal_dark\": \"-10%\",       \"panoramic_view_extra\": \"+5%\"     }   },   \"confidence_rules_logic\": {     \"boosts\": [       \"Zona specifica nota (Brera vs Milano generico). Nel testo: differenze tra zone > 8500 €/mq.\",       \"Dati reali inseriti (comparabili / affitti reali) → riduce errore.\",       \"Input completo (mq, piano, ascensore, stato, esposizione, ecc.).\"     ],     \"penalties\": [       \"Info mancanti (piano/ascensore/stato) → rischi errori tipo ~10% (stato) o ~5% (ascensore).\",       \"Zona generica 'Altro Milano' → range ampio (esempio varianza 2k–11k €/mq nel testo).\",       \"Pochi comparabili → confidenza più bassa.\"     ]   },   \"missing_blocks_to_complete_later\": [     \"Rendite long term specifiche per taglio (mono/bilo/trilo) per zona (da consolidare).\",     \"Rendite short term affidabili per zona: ADR + occupancy + stagionalità (da consolidare).\",     \"Liste comparabili reali per ogni zona.\",     \"Costi ristrutturazione 2025/2026 consolidati (range reali).\",     \"Regole di aggiustamento basate su dati reali (piano/ascensore/stato/esposizione).\",     \"Costi acquisto % 'range realistico' fissato per casi tipici.\"   ] }",
              "type": "string"
            }
          ]
        },
        "options": {}
      },
      "type": "n8n-nodes-base.set",
      "typeVersion": 3.4,
      "position": [
        416,
        144
      ],
      "id": "feba8697-31e2-4712-b1aa-a6aa620218a1",
      "name": "Edit Fields2"
    },
    {
      "parameters": {
        "mode": "combine",
        "combineBy": "combineByPosition",
        "numberInputs": 3,
        "options": {
          "includeUnpaired": false
        }
      },
      "type": "n8n-nodes-base.merge",
      "typeVersion": 3.2,
      "position": [
        768,
        0
      ],
      "id": "bf8fb873-8ee8-40d5-b52d-4265d47d54d4",
      "name": "Merge"
    },
    {
      "parameters": {
        "jsCode": "const raw = $json.similar_links || \"\";\n\nif (!raw || typeof raw !== \"string\" || raw.trim() === \"\") {\n  return [{\n    json: {\n      comparables_summary: {\n        method: \"user_text_parse_v1\",\n        count: 0,\n        notes: \"Nessun comparabile fornito dall’utente.\"\n      }\n    }\n  }];\n}\n\nconst priceRegex = /€\\s*([\\d\\.]+)/g;\nconst sqmRegex = /(\\d{2,4})\\s*(m²|mq|m2)/gi;\n\nconst prices = [];\nconst sqms = [];\n\nlet m;\nwhile ((m = priceRegex.exec(raw)) !== null) {\n  const n = Number(m[1].replace(/\\./g, \"\"));\n  if (Number.isFinite(n)) prices.push(n);\n}\n\nwhile ((m = sqmRegex.exec(raw)) !== null) {\n  const n = Number(m[1]);\n  if (Number.isFinite(n)) sqms.push(n);\n}\n\nconst ppsqm = [];\nfor (let i = 0; i < Math.min(prices.length, sqms.length); i++) {\n  ppsqm.push(prices[i] / sqms[i]);\n}\n\nconst median = arr => {\n  if (!arr.length) return null;\n  const s = [...arr].sort((a,b)=>a-b);\n  const mid = Math.floor(s.length/2);\n  return s.length % 2 ? s[mid] : (s[mid-1] + s[mid]) / 2;\n};\n\nconst median_ppsqm = median(ppsqm);\nconst count = ppsqm.length;\n\nlet confidence_boost = 0;\nif (count === 1) confidence_boost = 0.07;\nif (count === 2) confidence_boost = 0.10;\nif (count >= 3) confidence_boost = 0.12;\n\nreturn [{\n  json: {\n    comparables_summary: {\n      method: \"user_text_parse_v1\",\n      count,\n      median_ppsqm,\n      confidence_boost,\n      extracted_ppsqm: ppsqm,\n      notes: \"Comparabili estratti dal testo fornito dall’utente.\"\n    }\n  }\n}];\n"
      },
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        320,
        -176
      ],
      "id": "082c03bc-1207-456b-a7a5-244105812f0d",
      "name": "Code in JavaScript4"
    },
    {
      "parameters": {
        "jsCode": "const out = { ...$json };\n\nif (typeof out.benchmark_milano_v1 === \"string\") {\n  try {\n    out.benchmark_milano_v1 = JSON.parse(out.benchmark_milano_v1);\n    out.benchmark_parse_status = \"ok\";\n  } catch (e) {\n    out.benchmark_parse_status = \"error\";\n    out.benchmark_parse_error = e.message;\n  }\n} else {\n  out.benchmark_parse_status = \"already_object\";\n}\n\nreturn [{ json: out }];\n"
      },
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        560,
        144
      ],
      "id": "6d9ca99b-5368-4ac8-ad25-20b09220e5e1",
      "name": "Code in JavaScript5"
    },
    {
      "parameters": {
        "jsCode": "// Merge all incoming items into one object (NULL-SAFE)\nconst merged = {};\n\nfor (const item of $items()) {\n  const obj = item.json || {};\n\n  // ✅ Non sovrascrivere mai un valore buono con null/undefined\n  for (const k of Object.keys(obj)) {\n    if (obj[k] === null || obj[k] === undefined) continue;\n    merged[k] = obj[k];\n  }\n}\n\n// 🔴 PARSE BENCHMARK SE È STRINGA\nif (typeof merged.benchmark_milano_v1 === \"string\") {\n  try {\n    merged.benchmark_milano_v1 = JSON.parse(merged.benchmark_milano_v1);\n    merged.benchmark_parse_status = \"parsed_ok\";\n  } catch (e) {\n    merged.benchmark_parse_status = \"parse_error\";\n    merged.benchmark_parse_error = e.message;\n  }\n} else if (typeof merged.benchmark_milano_v1 === \"object\" && merged.benchmark_milano_v1 !== null) {\n  merged.benchmark_parse_status = \"already_object\";\n} else {\n  merged.benchmark_parse_status = \"missing_or_invalid\";\n}\n\n// ✅ DEBUG PRE-ENGINE (verifica cosa arriva al motore)\nmerged.__debug_pre_engine = {\n  has_benchmark: !!merged.benchmark_milano_v1,\n  benchmark_type: typeof merged.benchmark_milano_v1,\n  benchmark_parse_status: merged.benchmark_parse_status || null,\n  has_comps: !!merged.comparables_summary,\n  comps_type: typeof merged.comparables_summary,\n  comps_count: merged.comparables_summary?.count ?? null,\n  comps_median_ppsqm: merged.comparables_summary?.median_ppsqm ?? null,\n};\n\n// ✅ WARNING se i comparables spariscono (possibile overwrite nel merge)\nif (merged.comparables_summary == null) {\n  merged.__warning = \"comparables_summary missing after merge (possible overwrite)\";\n}\n\nreturn [{ json: merged }];\n\n\n\n"
      },
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        976,
        0
      ],
      "id": "e687bc14-6f83-4cd2-896d-b3207405dbb2",
      "name": "Code in JavaScript6"
    },
    {
      "parameters": {
        "jsCode": "// n8n Code node (JavaScript)\n// Guard \"blocked/zero report\" based on email_body text\n// Works on ALL items (map), because you often have 2 items in input.\n\nfunction cleanEmailBody(v) {\n  return String(v || \"\")\n    .replace(/\\r/g, \"\")\n    .trim();\n}\n\nfunction isBlockedBody(body) {\n  const b = body.toLowerCase();\n\n  // hard signals\n  if (b.includes(\"report bloccato\")) return true;\n  if (b.includes(\"dati critici mancanti\") || b.includes(\"fuori range\")) return true;\n\n  // \"zero report\" signals\n  const hasMidZero = /mid:\\s*€0\\b/i.test(body);\n  const hasLowZero = /low:\\s*€0\\b/i.test(body);\n  const hasHighZero = /high:\\s*€0\\b/i.test(body);\n  const hasPriceZero = /prezzo richiesto:\\s*€0\\b/i.test(body);\n  const hasCapitalZero = /capitale investito stimato:\\s*€0\\b/i.test(body);\n\n  // if at least 2 of these are true => it's junk\n  const score = [hasMidZero, hasLowZero, hasHighZero, hasPriceZero, hasCapitalZero].filter(Boolean).length;\n  return score >= 2;\n}\n\nfunction extractOne(body, regex, fallback = \"—\") {\n  const m = body.match(regex);\n  return (m && m[1]) ? String(m[1]).trim() : fallback;\n}\n\nfunction buildBlockedBody(originalBody) {\n  // Try to extract a few fields from the existing body (best effort)\n  const address = extractOne(originalBody, /Immobile:\\s*\\n([^\\n]+)/i, \"—\");\n  const sqm = extractOne(originalBody, /–\\s*([0-9]+)\\s*mq/i, \"—\");\n  const stato = extractOne(originalBody, /Stato:\\s*([^\\n]+)/i, \"—\");\n\n  return [\n    \"REPORT PREBETA – Milano (BLOCCATO)\",\n    \"\",\n    `Immobile: ${address}`,\n    `Mq: ${sqm}  |  Stato: ${stato}`,\n    \"\",\n    \"Non posso generare una stima affidabile perché i dati inseriti sono incoerenti o fuori range.\",\n    \"\",\n    \"Cosa fare adesso:\",\n    \"- Inserisci una superficie realistica (es. >= 10 mq).\",\n    \"- Inserisci un prezzo richiesto valido (> 0).\",\n    \"- Ricontrolla stato immobile e zona.\",\n    \"\",\n    \"Appena i dati sono coerenti, ti genero il report completo.\",\n    \"\",\n    \"Disclaimer: stima indicativa, non è una perizia.\"\n  ].join(\"\\n\");\n}\n\nreturn items.map((item) => {\n  const j = item.json || {};\n  const body = cleanEmailBody(j.email_body);\n\n  if (isBlockedBody(body)) {\n    j.email_body = buildBlockedBody(body);\n\n    // opzionale: metto un subject più chiaro (puoi anche commentare questa riga)\n    // j.email_subject = (j.email_subject || \"Report pre-beta Milano\").replace(/\\n/g, \"\").trim() + \" (BLOCCATO)\";\n  }\n\n  return { json: j };\n});\n"
      },
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        2240,
        0
      ],
      "id": "04336695-2c59-4cc2-b9fe-e816a7769004",
      "name": "Code in JavaScript7"
    },
    {
      "parameters": {
        "jsCode": "// Adapter IN: Emergent -> formato input motore (schema \"zone_tag/surface_sqm/asking_price_eur...\")\n// FIX: i campi della webapp arrivano in $json.body\nconst j = $json.body ?? $json;\n\n// helper\nconst cleanStr = (v) => (v === null || v === undefined) ? null : String(v).trim();\nconst toNum = (v) => {\n  const s = cleanStr(v);\n  if (s === null || s === \"\") return null;\n  const n = Number(s.replace(\",\", \".\"));\n  return Number.isFinite(n) ? n : null;\n};\n\n// Macrozone: \"navigli\" -> \"Navigli\"\nconst titleCase = (s) => {\n  const t = cleanStr(s);\n  if (!t) return null;\n  return t.charAt(0).toUpperCase() + t.slice(1).toLowerCase();\n};\n\n// property_type: \"monolocale/bilocale/trilocale...\" -> \"mono/bilo/trilo/quadri\"\nconst mapPropertyType = (t) => {\n  const x = (cleanStr(t) || \"\").toLowerCase();\n  if ([\"mono\", \"monolocale\", \"studio\"].includes(x)) return \"mono\";\n  if ([\"bilo\", \"bilocale\"].includes(x)) return \"bilo\";\n  if ([\"trilo\", \"trilocale\"].includes(x)) return \"trilo\";\n  if ([\"quadri\", \"quadrilocale\", \"4+\", \"4plus\"].includes(x)) return \"quadri\";\n  return cleanStr(t); // fallback\n};\n\n// comps_links: array -> stringa con newline\nconst linksToString = (v) => {\n  if (Array.isArray(v)) return v.filter(Boolean).map(String).join(\"\\n\");\n  const s = cleanStr(v);\n  return s ?? null;\n};\n\n// floor: supporta numeri e stringhe tipo \"secondo\", \"3°\", \"piano terra\"\nconst floorToNumber = (v) => {\n  if (v === null || v === undefined) return null;\n  if (typeof v === \"number\") return Number.isFinite(v) ? v : null;\n\n  const s = String(v).trim().toLowerCase();\n\n  // prova parsing numero diretto (es: \"3\", \"3°\", \"2° piano\")\n  const digits = s.match(/\\d+/);\n  if (digits) {\n    const n = Number(digits[0]);\n    return Number.isFinite(n) ? n : null;\n  }\n\n  const map = {\n    \"terra\": 0,\n    \"piano terra\": 0,\n    \"t\": 0,\n    \"primo\": 1,\n    \"secondo\": 2,\n    \"terzo\": 3,\n    \"quarto\": 4,\n    \"quinto\": 5,\n    \"sesto\": 6,\n    \"settimo\": 7,\n    \"ottavo\": 8,\n    \"nono\": 9,\n    \"decimo\": 10,\n  };\n\n  return map[s] ?? null;\n};\n\n// stress: se arriva 0-10 -> 0-1\nconst normalizeStress01 = (v) => {\n  const n = toNum(v);\n  if (n === null) return 1;\n  return n > 1 ? Math.max(0, Math.min(1, n / 10)) : Math.max(0, Math.min(1, n));\n};\n\n// elevator: boolean o stringhe\nconst normalizeElevator = (v) => {\n  if (typeof v === \"boolean\") return v;\n  const s = (cleanStr(v) || \"\").toLowerCase();\n  if (!s) return false;\n  if ([\"si\", \"sì\", \"yes\", \"true\", \"1\"].includes(s)) return true;\n  if ([\"no\", \"false\", \"0\"].includes(s)) return false;\n  return true;\n};\n\n// ✅ PATCH: address_text non deve mai essere vuoto (altrimenti il motore blocca)\nconst addressText = (() => {\n  const a = cleanStr(j.location_text) ?? cleanStr(j.address_text) ?? cleanStr(j.address);\n  if (a) return a;\n  const z = titleCase(j.macrozone);\n  return z ? `${z}, Milano` : \"Milano\";\n})();\n\nconst out = {\n  email: cleanStr(j.email) ?? null,\n  address_text: addressText,\n\n  zone_tag: titleCase(j.macrozone) ?? cleanStr(j.zone_tag) ?? null,\n  property_type: mapPropertyType(j.property_type),\n\n  surface_sqm: toNum(j.sqm) ?? toNum(j.surface_sqm),\n  floor: floorToNumber(j.floor),\n\n  condition: cleanStr(j.condition) ?? \"habitable\",\n  asking_price_eur: toNum(j.asking_price) ?? toNum(j.asking_price_eur),\n\n  expected_renovation_level: cleanStr(j.renovation_level) ?? cleanStr(j.expected_renovation_level) ?? null,\n\n  renovation_budget_override_eur: toNum(j.renovation_budget_override_eur) ?? toNum(j.renovation_budget) ?? null,\n  rent_long_term_gross_eur: toNum(j.rent_long_estimate) ?? toNum(j.rent_long_term_gross_eur) ?? null,\n  rent_short_term_gross_eur: toNum(j.rent_short_estimate) ?? toNum(j.rent_short_term_gross_eur) ?? null,\n\n  stress_tolerance_0_1: normalizeStress01(j.stress_tolerance ?? j.stress_tolerance_0_1),\n\n  elevator: normalizeElevator(j.elevator),\n\n  similar_links: linksToString(j.comps_links) ?? linksToString(j.similar_links) ?? null,\n};\n\nreturn [{ json: out }];"
      },
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        208,
        0
      ],
      "id": "248812fd-239f-4b64-9cd9-72a7d82233d8",
      "name": "Code in JavaScript8"
    },
    {
      "parameters": {
        "jsCode": "// Adapter OUT: parse email_body -> JSON per webapp (force GREEN/YELLOW/RED in both decision + engine_decision)\nconst body = ($json.email_body || \"\").toString();\n\n// --- helpers parse ---\nconst euroFromLine = (label) => {\n  const re = new RegExp(label + \"\\\\s*:\\\\s*€\\\\s*([0-9\\\\.]+)\", \"i\");\n  const m = body.match(re);\n  if (!m) return null;\n  return Number(m[1].replace(/\\./g, \"\"));\n};\n\nconst euroInline = (label) => {\n  const re = new RegExp(label + \"\\\\s*:\\\\s*€\\\\s*([0-9\\\\.]+)\", \"i\");\n  const m = body.match(re);\n  if (!m) return null;\n  return Number(m[1].replace(/\\./g, \"\"));\n};\n\nconst pctFromROI = (label) => {\n  const re = new RegExp(label + \"\\\\s*ROI\\\\s*:\\\\s*([0-9]+(?:\\\\.[0-9]+)?)%\", \"i\");\n  const m = body.match(re);\n  return m ? Number(m[1]) : null;\n};\n\nconst confidenceFromText = () => {\n  const m = body.match(/Confidenza\\s*:\\s*([0-9]+)\\s*%/i);\n  return m ? Number(m[1]) : null;\n};\n\nconst decisionFromText = () => {\n  const m = body.match(/Decisione finale\\s*:\\s*(GO|VERIFY|NO_GO|NO GO)/i);\n  if (!m) return \"VERIFY\";\n  return m[1].replace(\" \", \"_\").toUpperCase();\n};\n\nconst capitalFromText = () => {\n  const m = body.match(/Capitale investito stimato\\s*:\\s*€\\s*([0-9\\.]+)/i);\n  return m ? Number(m[1].replace(/\\./g, \"\")) : null;\n};\n\nconst renovationFromText = () => {\n  const m = body.match(/Ristrutturazione stimata\\s*:\\s*€\\s*([0-9\\.]+)/i);\n  return m ? Number(m[1].replace(/\\./g, \"\")) : null;\n};\n\nconst reasonsFromMotivation = () => {\n  const m = body.match(/Motivazione:\\s*([\\s\\S]*?)\\n\\nNota:/i);\n  const text = m ? m[1].trim() : \"\";\n  if (!text) return [];\n  const parts = text.split(/\\. /).map(s => s.trim()).filter(Boolean);\n  return parts.slice(0, 3);\n};\n\n// Scostamento vs mercato (mid): €312.814 (75.8%)\nconst spreadPctFromText = () => {\n  const m = body.match(/Scostamento vs mercato \\(mid\\)\\s*:\\s*€\\s*[0-9\\.]+\\s*\\(\\s*([0-9]+(?:\\.[0-9]+)?)%\\s*\\)/i);\n  return m ? Number(m[1]) : null;\n};\n\n// --- compute deal classification ---\nconst raw_engine_decision = decisionFromText();\nconst market_mid = euroFromLine(\"Mid\");\nconst asking = euroInline(\"Prezzo richiesto\");\nconst confidence = confidenceFromText();\nconst spread_pct = spreadPctFromText();\n\nconst computed_spread_pct = (() => {\n  if (spread_pct !== null) return spread_pct;\n  if (!market_mid || !asking) return null;\n  const diff = market_mid - asking;\n  return (diff / market_mid) * 100;\n})();\n\nconst deal_classification = (() => {\n  const s = computed_spread_pct;\n  if (s === null) return \"YELLOW\";\n  if (s >= 15) return \"GREEN\";\n  if (s >= 5) return \"YELLOW\";\n  return \"RED\";\n})();\n\n// 🔥 Forziamo i campi che la UI sta leggendo\nconst out = {\n  // la UI legge decision? -> GREEN/YELLOW/RED\n  decision: deal_classification,\n\n  // la UI legge engine_decision? -> GREEN/YELLOW/RED\n  engine_decision: deal_classification,\n\n  // teniamo comunque la decisione vera del motore (per debug)\n  raw_engine_decision,\n\n  deal_classification,\n  spread_pct: computed_spread_pct,\n  asking_price_eur: asking,\n\n  market_value_low: euroFromLine(\"Low\"),\n  market_value_mid: market_mid,\n  market_value_high: euroFromLine(\"High\"),\n\n  roi_long: pctFromROI(\"Affitto lungo\"),\n  roi_short: pctFromROI(\"Affitto breve\"),\n\n  confidence_percent: confidence,\n  key_reasons: reasonsFromMotivation(),\n\n  costs_breakdown: {\n    capital_invested: capitalFromText(),\n    renovation_estimated: renovationFromText(),\n  }\n};\n\nreturn [{ json: out }];"
      },
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        1824,
        0
      ],
      "id": "09624348-5b42-4bad-8cdf-83124bc96d79",
      "name": "Code in JavaScript"
    }
  ],
  "pinData": {},
  "connections": {
    "Webhook": {
      "main": [
        [
          {
            "node": "Code in JavaScript8",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Code in JavaScript1": {
      "main": [
        [
          {
            "node": "Code in JavaScript2",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Code in JavaScript2": {
      "main": [
        [
          {
            "node": "Code in JavaScript",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Edit Fields1": {
      "main": [
        [
          {
            "node": "Code in JavaScript7",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Message a model": {
      "main": [
        [
          {
            "node": "Code in JavaScript3",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Code in JavaScript3": {
      "main": [
        [
          {
            "node": "Send a message",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Merge": {
      "main": [
        [
          {
            "node": "Code in JavaScript6",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Edit Fields2": {
      "main": [
        [
          {
            "node": "Code in JavaScript5",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Code in JavaScript4": {
      "main": [
        [
          {
            "node": "Merge",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Code in JavaScript5": {
      "main": [
        [
          {
            "node": "Merge",
            "type": "main",
            "index": 2
          }
        ]
      ]
    },
    "Code in JavaScript6": {
      "main": [
        [
          {
            "node": "Code in JavaScript1",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Code in JavaScript7": {
      "main": [
        [
          {
            "node": "Message a model",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Code in JavaScript8": {
      "main": [
        [
          {
            "node": "Merge",
            "type": "main",
            "index": 1
          },
          {
            "node": "Code in JavaScript4",
            "type": "main",
            "index": 0
          },
          {
            "node": "Edit Fields2",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Code in JavaScript": {
      "main": [
        []
      ]
    }
  },
  "active": true,
  "settings": {
    "executionOrder": "v1",
    "binaryMode": "separate",
    "availableInMCP": false
  },
  "versionId": "e11c6f7b-4a8c-40f8-a5ad-0188f99aca32",
  "meta": {
    "templateCredsSetupCompleted": true,
    "instanceId": "11f68d44ee224fdcb9df4260f2f96792d3b8460a1663831026f5fc116a0979fc"
  },
  "id": "lFGjQ7orfdn3RnX6T2TjX",
  "tags": []
}
