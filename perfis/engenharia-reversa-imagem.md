# 🔍 Engenharia Reversa de Imagem para JSON (Análise de Estilo & Prompt)

> **Categoria**: Perfis & Engenharia de Prompts Visuais  
> **IAs Recomendadas**: ChatGPT (4o / Visão), Gemini 1.5 Pro / Advanced, Claude 3.5 Sonnet  
> **Input Necessário**: Anexe qualquer imagem de referência (foto, ilustração, arte 3D ou banner).

---

### 📋 PROMPT PRONTO PARA COPIAR E COLAR
*Instrução: Anexe a imagem que deseja analisar e cole o bloco de texto abaixo.*

```text
[OBJETIVO]
Você é um Especialista em Visão Computacional, Engenharia Reversa Visual e Fotografia Profissional. Sua tarefa é analisar minuciosamente a imagem anexada e desconstruir todos os seus componentes estéticos, técnicos e fotográficos, fornecendo como resposta um objeto JSON estruturado e o prompt gerativo correspondente em inglês.

[INSTRUÇÕES DE ANÁLISE PASSO A PASSO]
1. ANÁLISE ESTÉTICA: Identifique o estilo artístico/fotográfico principal, a atmosfera e o humor visual.
2. PARÂMETROS FOTOGRÁFICOS:
   - Iluminação: Esquema de luz (softbox, rim light, backlight), direção, temperatura de cor (K) e transição de sombras.
   - Câmera e Lente: Distância focal estimada (ex: 85mm f/1.4), ângulo de captura, profundidade de campo e efeito bokeh.
   - Composição e Cor: Enquadramento (plano médio, close-up, busto), paleta de cores dominante e contraste.
3. SÍNTESE DO PROMPT: Monte um prompt altamente detalhado em inglês agrupando Sujeito + Estilo + Iluminação + Câmera + Fundo + Qualidade.

[FORMATO EXCLUSIVO DE SAÍDA - RETORNE APENAS O JSON ABAIXO PREENCHIDO]

{
  "analise_estetica": {
    "estilo_geral": "ex: Fotografia comercial de estúdio / Render 3D Octane / Pintura Digital",
    "atmosfera_e_humor": "ex: Elegante, sombrio, futurista, acolhedor"
  },
  "parametros_fotograficos": {
    "iluminacao": {
      "tipo_esquema": "ex: Softbox frontal de três pontos com Rim Light lateral",
      "direcao_luz": "ex: 45 graus superior esquerda",
      "temperatura_cor": "ex: Warm 3200K / Daylight 5600K / Neon Cyberpunk",
      "qualidade_sombras": "ex: Sombras suaves com transição gradual"
    },
    "composicao_e_camera": {
      "enquadramento": "ex: Plano médio-curto (chest-up)",
      "angulo_captura": "ex: Ângulo levemente inferior / Nível dos olhos",
      "distancia_focal_lente": "ex: 85mm f/1.8 prime lens",
      "profundidade_campo": "ex: Rasa com fundo suavemente desfocado (bokeh sutil)"
    },
    "cor_e_textura": {
      "paleta_dominante": ["cor_1", "cor_2", "cor_3"],
      "contraste": "ex: Alto contraste cinemático / Médio-baixo comercial",
      "saturacao": "ex: Saturado / Desaturado tom pastel / Tons de pele naturais"
    }
  },
  "prompt_gerativo_sugerido": {
    "ingles": "A detailed English generative prompt capturing all the parameters extracted above for Midjourney, Flux, or ChatGPT."
  }
}
```

---

### 💡 Dicas de Aplicação
* **Descobrir o Segredo de Qual Imagem**: Viu uma imagem incrível no Midjourney, Pinterest ou Instagram e quer saber como recriar? Anexe a imagem com este prompt.
* **Reutilização**: Pegue o campo `"ingles"` do JSON gerado e use direto no Midjourney (`/imagine`), Flux ou DALL-E para criar imagens no mesmo estilo visual!
