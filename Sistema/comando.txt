#!/bin/zsh

emulate zsh
setopt no_nomatch
unsetopt xtrace
export LC_ALL=C
umask 077

MANAGER_BUILD="4.3.0"
MANAGER_DATA="20/08/2026"
MANAGER_FIRMA="8a16125d605d2ac22c04ca15334057b17b65d5030998edaaa9815b2073c34501"

self="${0:A}"
xattr -d com.apple.quarantine "$self" 2>/dev/null || true

qui="${0:A:h}"
nomequi="${qui:t}"
nomequi="${(L)nomequi}"

if [[ "$nomequi" == "strumenti" ]]; then
    base="${qui:h}"
else
    base="$qui"
fi

trovacartella() {
    local b="$1" n="$2" d x y
    y="${(L)n}"
    for d in "$b"/*(N/); do
        x="${d:t}"
        x="${(L)x}"
        if [[ "$x" == "$y" ]]; then
            print -r -- "$d"
            return 0
        fi
    done
    print -r -- "$b/$n"
}

versioni="$(trovacartella "$base" Versioni)"
archivio="$(trovacartella "$base" Archivio)"
strumenti="$(trovacartella "$base" Strumenti)"
temporanei="$strumenti/Temporanei"
sistema="$base/Sistema"
backupcomando="$sistema/comando.txt"
integritafile="$sistema/integrita.txt"
runtimecopy=""
catalogo="$strumenti/catalogo.txt"
stato="$strumenti/stato.txt"
blocco="$strumenti/blocco.txt"
compatibilita="$strumenti/compatibilita.txt"
webcache="$strumenti/web.txt"
correnteweb="$strumenti/corrente.txt"
lockdir="$strumenti/Avvio"
logroot="$HOME/Library/Logs/Roblox"
cdnbase="https://setup.rbxcdn.com/mac"
storicourl="$cdnbase/DeployHistory.txt"
resolverurl="https://setup-rbxcdn.github.io/mac/DeployHistory.txt"

riparadirinterno() {
    local p="$1"
    if [[ -L "$p" ]]; then
        rm -f "$p" 2>/dev/null || return 1
    elif [[ -e "$p" && ! -d "$p" ]]; then
        rm -f "$p" 2>/dev/null || return 1
    fi
    mkdir -p "$p" 2>/dev/null || return 1
    return 0
}

riparafileinterno() {
    local f="$1"
    if [[ -L "$f" ]]; then
        rm -f "$f" 2>/dev/null || return 1
    elif [[ -e "$f" && ! -f "$f" ]]; then
        rm -rf "$f" 2>/dev/null || return 1
    fi
    [[ -f "$f" ]] || : > "$f" 2>/dev/null || return 1
    return 0
}

for p in "$versioni" "$strumenti" "$temporanei" "$sistema"; do
    riparadirinterno "$p" || { print -r -- "Struttura interna non sicura."; exit 73; }
done
chmod 700 "$strumenti" "$temporanei" "$sistema" 2>/dev/null || true
for f in "$catalogo" "$stato" "$blocco" "$compatibilita" "$webcache" "$correnteweb" "$backupcomando" "$integritafile"; do
    riparafileinterno "$f" || { print -r -- "File interno non sicuro."; exit 73; }
done
runtimecopy="$(mktemp -t robloxstudio 2>/dev/null)"
if [[ -z "$runtimecopy" || ! -f "$runtimecopy" || -L "$runtimecopy" ]]; then
    print -r -- "Impossibile creare un file temporaneo sicuro."
    exit 73
fi

[[ -L "$lockdir" ]] && rm -f "$lockdir" 2>/dev/null
if ! mkdir "$lockdir" 2>/dev/null; then
    vecchio="$(cat "$lockdir/pid" 2>/dev/null)"
    if [[ "$vecchio" == "$$" ]]; then
        :
    elif [[ "$vecchio" == <-> ]] && kill -0 "$vecchio" 2>/dev/null; then
        print -r -- "Roblox Studio e gia aperto in un'altra finestra del Terminale."
        exit 1
    else
        rm -rf "$lockdir" 2>/dev/null
        mkdir -p "$lockdir" 2>/dev/null
    fi
fi
print -r -- "$$" > "$lockdir/pid" 2>/dev/null

if [[ -t 1 ]]; then
    reset=$'\033[0m'
    bold=$'\033[1m'
    red=$'\033[31m'
    green=$'\033[32m'
    yellow=$'\033[33m'
    blue=$'\033[34m'
    cyan=$'\033[36m'
    gray=$'\033[90m'
else
    reset=""
    bold=""
    red=""
    green=""
    yellow=""
    blue=""
    cyan=""
    gray=""
fi

LARG=80
RIGHE=24
BOX=76
MARG=2
INT=72
SPIN_PID=""
SPIN_ATTIVO=0
SPIN_MSG=""
GUARD_PID=""
LOGPID=""
FINESTRA=""
BOUNDS=""
SISTEMA=""
ARCHMAC=""
DATA=""
VERS=""
VERIFICA_TTL=43200
OOD_TTL=86400
FUNZIONALI_SESSIONE=8
RIDIMENSIONATA=0

larghezzaterminale() {
    local dim c=0 r=0
    if [[ -t 0 ]]; then
        dim="$(stty size 2>/dev/null)"
        if [[ -n "$dim" ]]; then
            r="${dim%% *}"
            c="${dim##* }"
        fi
    fi
    if [[ "$c" != <-> ]] || (( c <= 0 )); then
        c="$(tput cols 2>/dev/null)"
        r="$(tput lines 2>/dev/null)"
    fi
    [[ "$c" == <-> ]] || c=80
    [[ "$r" == <-> ]] || r=24
    (( c <= 0 )) && c=80
    (( r <= 0 )) && r=24
    print -r -- "$c $r"
}

aggiornalayout() {
    local dim reale
    dim="$(larghezzaterminale)"
    LARG="${dim%% *}"
    RIGHE="${dim##* }"

    [[ "$LARG" == <-> ]] || LARG=80
    [[ "$RIGHE" == <-> ]] || RIGHE=24
    (( LARG <= 0 )) && LARG=80
    (( RIGHE <= 0 )) && RIGHE=24

    if (( LARG >= 48 )); then
        BOX=$(( LARG - 2 ))
    else
        BOX=$LARG
    fi

    (( BOX > 104 )) && BOX=104
    (( BOX > LARG )) && BOX=$LARG
    (( BOX < 8 )) && BOX=8

    MARG=$(( (LARG - BOX) / 2 ))
    (( MARG < 0 )) && MARG=0

    INT=$(( BOX - 4 ))
    (( INT < 4 )) && INT=4
}


TRAPWINCH() {
    RIDIMENSIONATA=1
}


ripeti() {
    local c="$1" n="$2"
    (( n <= 0 )) && return 0
    printf '%*s' "$n" '' | tr ' ' "$c"
}

margine() {
    (( MARG > 0 )) && printf '%*s' "$MARG" ''
}

tronca() {
    local t="$1" l="$2"
    (( l <= 0 )) && return 0
    if (( ${#t} > l )); then
        if (( l == 1 )); then
            print -rn -- "${t[1,1]}"
        else
            print -rn -- "${t[1,$(( l - 1 ))]}>"
        fi
    else
        print -rn -- "$t"
    fi
}

schermata() {
    [[ -t 1 ]] && printf '\033[2J\033[3J\033[H'
}

bordo() {
    local col="$1"
    margine
    print -rn -- "${col}+"
    ripeti '-' $(( BOX - 2 ))
    print -r -- "+${reset}"
}

riga() {
    local t="$1" col="$2" al="$3" sx dx spazio
    t="$(tronca "$t" "$INT")"
    margine
    if [[ "$al" == "centro" ]]; then
        spazio=$(( INT - ${#t} ))
        sx=$(( spazio / 2 ))
        dx=$(( spazio - sx ))
        printf '%s|%s ' "$col" "$reset"
        printf '%*s' "$sx" ''
        printf '%s%s%s' "$bold" "$t" "$reset"
        printf '%*s' "$dx" ''
        printf ' %s|%s\n' "$col" "$reset"
    else
        printf '%s|%s %s%-*s%s %s|%s\n' "$col" "$reset" "$bold" "$INT" "$t" "$reset" "$col" "$reset"
    fi
}

titolo() {
    aggiornalayout
    RIDIMENSIONATA=0
    schermata
    print
    bordo "$cyan"
    riga "Roblox Studio" "$cyan" centro
    riga "Manager $MANAGER_BUILD - $MANAGER_DATA" "$gray" centro
    bordo "$cyan"
}

sezione() {
    print
    bordo "$blue"
    riga "$1" "$blue" sinistra
    bordo "$blue"
}

testo() {
    local col="$1"
    shift
    local t="$*" parola linea="" larg=$INT
    local -a parole
    (( larg < 4 )) && larg=4
    parole=(${=t})
    if (( ${#parole} == 0 )); then
        print
        return 0
    fi
    for parola in "${parole[@]}"; do
        while (( ${#parola} > larg )); do
            if [[ -n "$linea" ]]; then
                margine
                print -r -- "${col}${linea}${reset}"
                linea=""
            fi
            margine
            print -r -- "${col}${parola[1,$larg]}${reset}"
            parola="${parola[$(( larg + 1 )),-1]}"
        done
        if [[ -z "$linea" ]]; then
            linea="$parola"
        elif (( ${#linea} + 1 + ${#parola} <= larg )); then
            linea="$linea $parola"
        else
            margine
            print -r -- "${col}${linea}${reset}"
            linea="$parola"
        fi
    done
    if [[ -n "$linea" ]]; then
        margine
        print -r -- "${col}${linea}${reset}"
    fi
}

info() { testo "" "$@" }
ok() { testo "$green" "$@" }
errore() { testo "$red" "$@" }
avviso() { testo "$yellow" "$@" }
nota() { testo "$gray" "$@" }

campo() {
    local k="$1" v="$2" kl=18
    v="$(formatovisibile "$v")"
    (( kl > INT / 2 )) && kl=$(( INT / 2 ))
    margine
    printf '%-*s ' "$kl" "$(tronca "$k" "$kl")"
    print -r -- "$(tronca "$v" $(( INT - kl - 1 )))"
}

voce() {
    local n="$1" t="$2"
    margine
    printf ' %s%s%s  %s\n' "$bold" "$n" "$reset" "$(tronca "$t" $(( INT - 5 )))"
}

domanda() {
    local etichetta="$1"
    margine
    printf '%s ' "$etichetta"
}

pausa() {
    local x
    print
    margine
    printf '%sInvio per continuare%s ' "$gray" "$reset"
    IFS= read -r x || return 0
}

colorecella() {
    case "$1" in
        (valida|compatibile|disponibile|online|si|bloccata|OK|CORRENTE|RECENTE|AVVIABILE) print -rn -- "$green" ;;
        (OUT*|obsoleta*|"non avviabile"|"macos vecchio"|"cpu"|offline|assente|"non disponibile"|"NON AVVIABILE") print -rn -- "$red" ;;
        (incerta*|"da verificare"|"DA TESTARE"|INCERTA|sconosciuta|"non verificata"|"hash nascosto"|"rete incerta"|VECCHIA) print -rn -- "$yellow" ;;
        (libera|PRECEDENTE) print -rn -- "$cyan" ;;
        (no|-|STORICA) print -rn -- "$gray" ;;
        (*) print -rn -- "" ;;
    esac
}

TTIT=()
TMIN=()
TPRE=()
TPRI=()
TON=()
TW=()

tabinit() {
    local s t mn pf pr i n avail somma peggiore resto quota
    TTIT=()
    TMIN=()
    TPRE=()
    TPRI=()
    TON=()
    TW=()
    for s in "$@"; do
        t="${s%%|*}"
        s="${s#*|}"
        mn="${s%%|*}"
        s="${s#*|}"
        pf="${s%%|*}"
        pr="${s#*|}"
        TTIT+=("$t")
        TMIN+=("$mn")
        TPRE+=("$pf")
        TPRI+=("$pr")
        TON+=(1)
        TW+=("$mn")
    done
    n=${#TTIT}
    avail=$(( BOX - 1 ))
    while true; do
        somma=0
        for (( i=1; i<=n; i++ )); do
            (( TON[i] == 1 )) && (( somma += TMIN[i] + 3 ))
        done
        (( somma <= avail )) && break
        peggiore=0
        for (( i=1; i<=n; i++ )); do
            (( TON[i] == 1 )) || continue
            (( TPRI[i] == 0 )) && continue
            if (( peggiore == 0 )) || (( TPRI[i] > TPRI[peggiore] )); then
                peggiore=$i
            fi
        done
        (( peggiore == 0 )) && break
        TON[$peggiore]=0
    done
    while true; do
        somma=0
        for (( i=1; i<=n; i++ )); do
            (( TON[i] == 1 )) && (( somma += TMIN[i] + 3 ))
        done
        (( somma <= avail )) && break
        peggiore=0
        for (( i=1; i<=n; i++ )); do
            (( TON[i] == 1 )) || continue
            (( TMIN[i] <= 3 )) && continue
            if (( peggiore == 0 )) || (( TMIN[i] > TMIN[peggiore] )); then
                peggiore=$i
            fi
        done
        (( peggiore == 0 )) && break
        TMIN[$peggiore]=$(( TMIN[peggiore] - 1 ))
        TW[$peggiore]=${TMIN[peggiore]}
    done
    while (( somma > avail )); do
        peggiore=0
        for (( i=n; i>=1; i-- )); do
            (( TON[i] == 1 )) || continue
            [[ "${TTIT[i]}" == "N" ]] && continue
            [[ "${TTIT[i]}" == "Versione" ]] && continue
            peggiore=$i
            break
        done
        (( peggiore == 0 )) && break
        TON[$peggiore]=0
        somma=0
        for (( i=1; i<=n; i++ )); do
            (( TON[i] == 1 )) && (( somma += TMIN[i] + 3 ))
        done
    done

    resto=$(( avail - somma ))
    (( resto < 0 )) && resto=0
    for (( i=1; i<=n; i++ )); do
        (( TON[i] == 1 )) || continue
        quota=$(( TPRE[i] - TMIN[i] ))
        (( quota <= 0 )) && continue
        (( quota > resto )) && quota=$resto
        TW[$i]=$(( TMIN[i] + quota ))
        (( resto -= quota ))
    done
    if (( resto > 0 )); then
        local flessibili=0 assegnato ultima=0
        for (( i=1; i<=n; i++ )); do
            (( TON[i] == 1 )) || continue
            ultima=$i
            (( TPRE[i] > TMIN[i] )) && (( flessibili++ ))
        done
        if (( flessibili > 0 )); then
            while (( resto > 0 )); do
                assegnato=0
                for (( i=1; i<=n; i++ )); do
                    (( resto > 0 )) || break
                    (( TON[i] == 1 )) || continue
                    (( TPRE[i] > TMIN[i] )) || continue
                    TW[$i]=$(( TW[i] + 1 ))
                    (( resto-- ))
                    assegnato=1
                done
                (( assegnato == 0 )) && break
            done
        elif (( ultima > 0 )); then
            TW[$ultima]=$(( TW[ultima] + resto ))
        fi
    fi
}

tabbordo() {
    local i
    margine
    print -rn -- "+"
    for (( i=1; i<=${#TTIT}; i++ )); do
        (( TON[i] == 1 )) || continue
        ripeti '-' $(( TW[i] + 2 ))
        print -rn -- "+"
    done
    print
}

tabtesta() {
    local i
    margine
    print -rn -- "|"
    for (( i=1; i<=${#TTIT}; i++ )); do
        (( TON[i] == 1 )) || continue
        printf ' %s%-*s%s |' "$bold" "${TW[i]}" "$(tronca "${TTIT[i]}" "${TW[i]}")" "$reset"
    done
    print
}

tabriga() {
    local -a v
    local i c t originale
    v=("$@")
    margine
    print -rn -- "|"
    for (( i=1; i<=${#TTIT}; i++ )); do
        (( TON[i] == 1 )) || continue
        t="${v[i]}"
        [[ -z "$t" ]] && t="-"
        originale="$t"
        t="$(formatovisibile "$t")"
        c="$(colorecella "$originale")"
        printf ' %s%-*s%s |' "$c" "${TW[i]}" "$(tronca "$t" "${TW[i]}")" "$reset"
    done
    print
}

spinnerferma() {
    SPIN_PID=""
    SPIN_ATTIVO=0
    SPIN_MSG=""
    if [[ -t 1 ]]; then
        printf '\r\033[2K\033[?25h'
    fi
}

spinneravvia() {
    local msg="$1" lim
    spinnerferma
    if [[ ! -t 1 ]]; then
        return 0
    fi
    if (( RIDIMENSIONATA == 1 )); then
        aggiornalayout
        RIDIMENSIONATA=0
    fi
    lim=$(( INT - 7 ))
    (( lim < 4 )) && lim=4
    msg="$(tronca "$msg" "$lim")"
    SPIN_ATTIVO=1
    SPIN_MSG="$msg"
    printf '\033[?25l\r\033[2K'
    margine
    printf '[....] %s' "$msg"
}


barra() {
    local pct="$1" testo="$2" larghezzabarra pieni vuoti linea
    if (( RIDIMENSIONATA == 1 )); then
        aggiornalayout
        RIDIMENSIONATA=0
    fi
    (( pct < 0 )) && pct=0
    (( pct > 100 )) && pct=100
    larghezzabarra=$(( INT - ${#testo} - 8 ))
    (( larghezzabarra > 40 )) && larghezzabarra=40
    (( larghezzabarra < 4 )) && larghezzabarra=4
    pieni=$(( pct * larghezzabarra / 100 ))
    vuoti=$(( larghezzabarra - pieni ))
    linea="[$(ripeti '#' $pieni)$(ripeti '.' $vuoti)] ${pct}% $testo"
    printf '\r\033[2K'
    margine
    printf '%s' "$(tronca "$linea" "$INT")"
}


megabyte() {
    local n="$1"
    [[ "$n" == <-> ]] || n=0
    printf '%.1f' $(( n / 1048576.0 ))
}

durata() {
    local s="$1" m
    [[ "$s" == <-> ]] || s=0
    m=$(( s / 60 ))
    s=$(( s % 60 ))
    printf '%d:%02d' "$m" "$s"
}

dimensionefile() {
    local n=""
    [[ -f "$1" ]] || { print -r -- 0; return 0 }
    n="$(stat -f%z "$1" 2>/dev/null)"
    if [[ "$n" != <-> ]]; then
        n="$(wc -c < "$1" 2>/dev/null | tr -d ' ')"
    fi
    [[ "$n" == <-> ]] || n=0
    print -r -- "$n"
}

conlimite() {
    local lim="$1"
    shift
    local p i=0 esito
    "$@" <&0 &
    p=$!
    while (( i < lim * 10 )); do
        kill -0 "$p" 2>/dev/null || break
        sleep 0.1
        (( i++ ))
    done
    if kill -0 "$p" 2>/dev/null; then
        kill -9 "$p" 2>/dev/null
        wait "$p" 2>/dev/null
        return 124
    fi
    wait "$p"
    esito=$?
    return $esito
}

adattafinestra() {
    aggiornalayout
}


ripristinafinestra() {
    return 0
}


uscita() {
    spinnerferma
    fermaguardia
    if [[ -n "$LOGPID" ]]; then
        kill "$LOGPID" 2>/dev/null
        LOGPID=""
    fi
    rm -f "$runtimecopy" 2>/dev/null || true
    rm -rf "$lockdir" 2>/dev/null
    ripristinafinestra
    [[ -t 1 ]] && printf '\033[?25h'
}


trap uscita EXIT
trap 'exit 130' INT TERM HUP

normalizzadata() {
    local valore="$1"
    if [[ "$valore" =~ '^([0-9]{4})-([0-9]{1,2})-([0-9]{1,2})$' ]]; then
        printf '%04d-%02d-%02d\n' "${match[1]}" "${match[2]}" "${match[3]}"
        return 0
    fi
    if [[ "$valore" =~ '^([0-9]{2})/([0-9]{2})/([0-9]{4})$' ]]; then
        printf '%04d-%02d-%02d\n' "${match[3]}" "${match[2]}" "${match[1]}"
        return 0
    fi
    print -r -- ""
}

formatovisibile() {
    local d="$1"
    if [[ "$d" =~ '^([0-9]{4})-([0-9]{2})-([0-9]{2}) ([0-9]{2}):([0-9]{2})(:([0-9]{2}))?$' ]]; then
        printf '%02d/%02d/%04d %02d:%02d\n' "${match[3]}" "${match[2]}" "${match[1]}" "${match[4]}" "${match[5]}"
        return 0
    fi
    if [[ "$d" =~ '^([0-9]{4})-([0-9]{2})-([0-9]{2})$' ]]; then
        printf '%02d/%02d/%04d\n' "${match[3]}" "${match[2]}" "${match[1]}"
        return 0
    fi
    print -r -- "$d"
}

datavisibile() {
    formatovisibile "$1"
}

pulisciriga() {
    tr -d '\r' | sed -e 's/[[:space:]]*$//'
}

hashvalido() {
    [[ "$1" =~ '^version-[0-9a-fA-F]{16}$' ]]
}

versionevalida() {
    [[ "$1" =~ '^0\.[0-9]+\.[0-9]+\.[0-9]+$' ]]
}

curlhttps() {
    command curl --proto '=https' --proto-redir '=https' "$@"
}

normalizzacatalogo() {
    local tmp="$temporanei/catalogo.tmp"
    [[ -s "$catalogo" ]] || return 0
    tr '|' '\t' < "$catalogo" | pulisciriga | awk -F '\t' 'BEGIN{OFS="\t"}
    NF>=3 {
        split($1,a,"-")
        if(a[1] ~ /^[0-9][0-9][0-9][0-9]$/ && a[2] ~ /^[0-9]+$/ && a[3] ~ /^[0-9]+$/){
            d=sprintf("%04d-%02d-%02d",a[1],a[2],a[3])
            v=$2
            h=$3
            gsub(/^[ \t]+|[ \t]+$/,"",v)
            gsub(/^[ \t]+|[ \t]+$/,"",h)
            if(v ~ /^0\.[0-9]+\.[0-9]+\.[0-9]+$/ && (h=="version-hidden" || (length(h)==24 && h ~ /^version-[0-9a-fA-F]+$/))) print d,v,h
        }
    }' | awk -F '\t' 'BEGIN{OFS="\t"}
    {
        k=$2
        if(!(k in linea) || (hash[k]=="version-hidden" && $3!="version-hidden")){
            linea[k]=$0
            hash[k]=$3
        }
    }
    END{
        for(k in linea) print linea[k]
    }' | sort -t $'\t' -k1,1r -k2,2r > "$tmp" 2>/dev/null
    if [[ -s "$tmp" ]]; then
        mv "$tmp" "$catalogo"
    else
        rm -f "$tmp"
    fi
}

normalizzastato() {
    local tmp="$temporanei/stato.tmp"
    [[ -s "$stato" ]] || return 0
    pulisciriga < "$stato" | awk -F '\t' 'BEGIN{OFS="\t"}
    NF>=3 {
        if($1 ~ /^[0-9][0-9][0-9][0-9]-[0-9][0-9]-[0-9][0-9]$/ && $2 ~ /^0\.[0-9]+\./){
            if($3=="supportata"||$3=="obsoleta"||$3=="incerta"||$3=="nonavviabile") print $1,$2,$3,$4,$5
        }
    }' | awk -F '\t' '!visti[$2]++' | sort -t $'\t' -k1,1r -k2,2r > "$tmp" 2>/dev/null
    mv "$tmp" "$stato" 2>/dev/null
}

normalizzacompatibilita() {
    local tmp="$temporanei/compat.tmp"
    [[ -s "$compatibilita" ]] || return 0
    pulisciriga < "$compatibilita" | awk -F '\t' 'BEGIN{OFS="\t"}
    NF>=6 {
        if($1 ~ /^[0-9][0-9][0-9][0-9]-[0-9][0-9]-[0-9][0-9]$/ && $2 ~ /^0\.[0-9]+\./) print $1,$2,$3,$4,$5,$6,$7
    }' | awk -F '\t' '!visti[$2]++' | sort -t $'\t' -k1,1r -k2,2r > "$tmp" 2>/dev/null
    mv "$tmp" "$compatibilita" 2>/dev/null
}

normalizzablocco() {
    local tmp="$temporanei/blocco.tmp"
    [[ -s "$blocco" ]] || return 0
    pulisciriga < "$blocco" | awk -F '\t' 'BEGIN{OFS="\t"}
    NF>=3 {
        if($2 ~ /^0\.[0-9]+\./) print $1,$2,$3
    }' | awk -F '\t' '!visti[$2]++' > "$tmp" 2>/dev/null
    mv "$tmp" "$blocco" 2>/dev/null
}

normalizzaweb() {
    local tmp="$temporanei/web.tmp"
    [[ -s "$webcache" ]] || return 0
    pulisciriga < "$webcache" | awk -F '\t' 'BEGIN{OFS="\t"}
    NF>=3 {
        if(length($1)==24 && $1 ~ /^version-[0-9a-fA-F]+$/ && $3 ~ /^[0-9]+$/) print $1,$2,$3
    }' | awk -F '\t' '!visti[$1]++' > "$tmp" 2>/dev/null
    mv "$tmp" "$webcache" 2>/dev/null
}

preparadati() {
    SISTEMA="$(sw_vers -productVersion 2>/dev/null)"
    ARCHMAC="$(uname -m 2>/dev/null)"
    [[ -n "$ARCHMAC" ]] || ARCHMAC="sconosciuta"
    normalizzacatalogo
    normalizzastato
    normalizzacompatibilita
    normalizzablocco
    normalizzaweb
}

firmafile() {
    local f="$1"
    [[ -f "$f" && ! -L "$f" ]] || return 1
    sed -E 's/^MANAGER_FIRMA="[^"]*"$/MANAGER_FIRMA=""/' "$f" 2>/dev/null | shasum -a 256 2>/dev/null | awk '{print $1}'
}

scriviintegrita() {
    local tmp="$sistema/integrita.tmp"
    mkdir -p "$sistema" 2>/dev/null || return 1
    printf '%s\t%s\t%s\n' "$MANAGER_BUILD" "$MANAGER_DATA" "$MANAGER_FIRMA" > "$tmp" 2>/dev/null || return 1
    mv "$tmp" "$integritafile" 2>/dev/null || return 1
    chmod 444 "$integritafile" 2>/dev/null || true
    return 0
}

copiaprotetta() {
    local fonte="$1" destinazione="$2" tmp
    [[ -f "$fonte" && ! -L "$fonte" ]] || return 1
    riparadirinterno "${destinazione:h}" || return 1
    tmp="${destinazione:h}/.${destinazione:t}.$$"
    rm -f "$tmp" 2>/dev/null
    /bin/cp -X "$fonte" "$tmp" 2>/dev/null || return 1
    chmod 444 "$tmp" 2>/dev/null || true
    mv -f "$tmp" "$destinazione" 2>/dev/null || { rm -f "$tmp" 2>/dev/null; return 1; }
    return 0
}

assicurasistema() {
    local fs fb

    [[ -d "$base" && ! -L "$base" ]] || return 1

    fs="$(firmafile "$self" 2>/dev/null)"

    if [[ "$fs" != "$MANAGER_FIRMA" ]]; then
        fb="$(firmafile "$backupcomando" 2>/dev/null)"
        if [[ "$fb" == "$MANAGER_FIRMA" ]]; then
            chmod u+w "$self" 2>/dev/null || true
            if /bin/cp -X "$backupcomando" "$self" 2>/dev/null; then
                chmod 555 "$self" 2>/dev/null || true
                xattr -d com.apple.quarantine "$self" 2>/dev/null || true
                print -r -- "Integrita ripristinata. Riavvio Roblox Studio."
                exec "$self"
            fi
        fi
        print -r -- "Integrita del manager non valida. Reinstalla robloxstudio.txt."
        exit 78
    fi

    riparadirinterno "$versioni" || return 1
    riparadirinterno "$strumenti" || return 1
    riparadirinterno "$temporanei" || return 1
    riparadirinterno "$sistema" || return 1
    chmod 700 "$strumenti" "$temporanei" "$sistema" 2>/dev/null || true

    riparafileinterno "$catalogo" || return 1
    riparafileinterno "$stato" || return 1
    riparafileinterno "$blocco" || return 1
    riparafileinterno "$compatibilita" || return 1
    riparafileinterno "$webcache" || return 1
    riparafileinterno "$correnteweb" || return 1
    [[ -L "$backupcomando" ]] && rm -f "$backupcomando" 2>/dev/null
    [[ -L "$integritafile" ]] && rm -f "$integritafile" 2>/dev/null

    fb="$(firmafile "$backupcomando" 2>/dev/null)"
    if [[ "$fb" != "$MANAGER_FIRMA" ]]; then
        copiaprotetta "$self" "$backupcomando" || true
    fi

    if [[ ! -f "$integritafile" ]] || ! grep -Fq "$MANAGER_FIRMA" "$integritafile" 2>/dev/null; then
        chmod u+w "$integritafile" 2>/dev/null || true
        scriviintegrita || true
    fi

    /bin/cp -X "$self" "$runtimecopy" 2>/dev/null || true
    chmod 400 "$runtimecopy" 2>/dev/null || true
    chmod 555 "$self" 2>/dev/null || true
    chmod 444 "$backupcomando" "$integritafile" 2>/dev/null || true
}

guardiasistema() {
    local fonte fs fb fr

    while true; do
        sleep 3
        [[ -d "$base" && ! -L "$base" ]] || return 0

        riparadirinterno "$versioni" || return 1
        riparadirinterno "$strumenti" || return 1
        riparadirinterno "$temporanei" || return 1
        riparadirinterno "$sistema" || return 1
        riparafileinterno "$catalogo" || return 1
        riparafileinterno "$stato" || return 1
        riparafileinterno "$blocco" || return 1
        riparafileinterno "$compatibilita" || return 1
        riparafileinterno "$webcache" || return 1
        riparafileinterno "$correnteweb" || return 1
        [[ -L "$backupcomando" ]] && rm -f "$backupcomando" 2>/dev/null
        [[ -L "$integritafile" ]] && rm -f "$integritafile" 2>/dev/null

        fs="$(firmafile "$self" 2>/dev/null)"
        fb="$(firmafile "$backupcomando" 2>/dev/null)"
        fr="$(firmafile "$runtimecopy" 2>/dev/null)"

        if [[ "$fs" != "$MANAGER_FIRMA" ]]; then
            fonte=""
            [[ "$fb" == "$MANAGER_FIRMA" ]] && fonte="$backupcomando"
            [[ -z "$fonte" && "$fr" == "$MANAGER_FIRMA" ]] && fonte="$runtimecopy"
            if [[ -n "$fonte" ]]; then
                mkdir -p "$strumenti" 2>/dev/null || true
                chmod u+w "$self" 2>/dev/null || true
                /bin/cp -X "$fonte" "$self" 2>/dev/null || true
                chmod 555 "$self" 2>/dev/null || true
                xattr -d com.apple.quarantine "$self" 2>/dev/null || true
            fi
        fi

        fb="$(firmafile "$backupcomando" 2>/dev/null)"
        if [[ "$fb" != "$MANAGER_FIRMA" ]]; then
            fonte=""
            [[ "$(firmafile "$self" 2>/dev/null)" == "$MANAGER_FIRMA" ]] && fonte="$self"
            [[ -z "$fonte" && "$fr" == "$MANAGER_FIRMA" ]] && fonte="$runtimecopy"
            [[ -n "$fonte" ]] && copiaprotetta "$fonte" "$backupcomando" || true
        fi

        if [[ ! -f "$integritafile" ]] || ! grep -Fq "$MANAGER_FIRMA" "$integritafile" 2>/dev/null; then
            chmod u+w "$integritafile" 2>/dev/null || true
            scriviintegrita || true
        fi
    done
}

avviaguardia() {
    if [[ -n "$GUARD_PID" ]] && kill -0 "$GUARD_PID" 2>/dev/null; then
        return 0
    fi
    guardiasistema >/dev/null 2>&1 &
    GUARD_PID=$!
}

fermaguardia() {
    if [[ -n "$GUARD_PID" ]]; then
        kill "$GUARD_PID" 2>/dev/null || true
        wait "$GUARD_PID" 2>/dev/null || true
        GUARD_PID=""
    fi
}

puliscistrumenti() {
    local file nome
    mkdir -p "$strumenti" "$temporanei" 2>/dev/null
    for file in "$strumenti"/*(N.); do
        nome="${file:t}"
        case "$nome" in
            (.DS_Store|aggiornaordine.command|ripararobloxstudio.command|unificarobloxstudio.command|aggiornacompatibilita.command|aggiornainterfaccia.command|aggiornapulizia.command|aggiornastrumenti.command|aggiornaverifica.command) rm -f "$file" 2>/dev/null ;;
            (*) ;;
        esac
    done
    rm -rf "$strumenti/.avvio" 2>/dev/null || true
}

pulisciprogetto() {
    local d

    find "$base" -maxdepth 3 -type f -name '.DS_Store' -delete 2>/dev/null

    for d in "$versioni"/*(N/); do
        if [[ -z "$(find "$d" -mindepth 1 -maxdepth 1 -print -quit 2>/dev/null)" ]]; then
            rmdir "$d" 2>/dev/null || true
        fi
    done

    if [[ -d "$archivio" && -z "$(find "$archivio" -mindepth 1 -maxdepth 1 -print -quit 2>/dev/null)" ]]; then
        rmdir "$archivio" 2>/dev/null || true
    fi

    if [[ -d "$temporanei/cdn" && -z "$(find "$temporanei/cdn" -mindepth 1 -maxdepth 1 -print -quit 2>/dev/null)" ]]; then
        rmdir "$temporanei/cdn" 2>/dev/null || true
    fi
}


versioneminore() {
    awk -v a="$1" -v b="$2" 'BEGIN{
        na=split(a,A,".")
        nb=split(b,B,".")
        n=na>nb?na:nb
        for(i=1;i<=n;i++){
            x=(i<=na?A[i]:0)+0
            y=(i<=nb?B[i]:0)+0
            if(x<y) exit 0
            if(x>y) exit 1
        }
        exit 1
    }'
}

reteattiva() {
    curlhttps -s --connect-timeout 3 --max-time 6 -o /dev/null -w '%{http_code}' "$storicourl" 2>/dev/null | grep -q '^[23]'
}

versionecorrente() {
    local now cached ctime tmp corrente
    now="$(date +%s)"
    cached=""
    ctime=0
    if [[ -s "$correnteweb" ]]; then
        IFS=$'\t' read -r cached ctime < "$correnteweb"
    fi
    [[ "$ctime" == <-> ]] || ctime=0
    if [[ -n "$cached" ]] && (( now - ctime < 1800 )); then
        print -r -- "$cached"
        return 0
    fi
    tmp="$temporanei/corrente.tmp"
    if curlhttps -fsSL --connect-timeout 4 --max-time 10 "$storicourl" -o "$tmp" 2>/dev/null; then
        corrente="$(grep '^New Studio ' "$tmp" 2>/dev/null | tail -n 1 | sed -E 's/.*git hash: ([0-9.]+).*/\1/')"
        rm -f "$tmp"
        if [[ "$corrente" == 0.* ]]; then
            printf '%s\t%s\n' "$corrente" "$now" > "$correnteweb"
            print -r -- "$corrente"
            return 0
        fi
    fi
    rm -f "$tmp" 2>/dev/null
    print -r -- "$cached"
}

estraistudio() {
    local ingresso="$1" uscita="$2"
    sed -nE 's/^New Studio (version-[^ ]+) at ([0-9]+)\/([0-9]+)\/([0-9]+).*git hash: ([0-9.]+).*$/\4-\2-\3\t\5\t\1/p' "$ingresso" | awk -F '\t' 'BEGIN{OFS="\t"}{split($1,a,"-");$1=sprintf("%04d-%02d-%02d",a[1],a[2],a[3]);print}' > "$uscita" 2>/dev/null
}

fondistorici() {
    local ufficiale="$1" risolto="$2" uscita="$3"
    local off="$temporanei/off.tsv" res="$temporanei/res.tsv"
    estraistudio "$ufficiale" "$off"
    if [[ -s "$risolto" ]]; then
        estraistudio "$risolto" "$res"
    else
        : > "$res"
    fi
    awk -F '\t' 'BEGIN{OFS="\t"}
    FNR==NR {
        if(length($3)==24 && $3 ~ /^version-[0-9a-fA-F]+$/) r[$1 SUBSEP $2]=$3
        next
    }
    {
        k=$1 SUBSEP $2
        if($3=="version-hidden" && (k in r)) $3=r[k]
        print
    }' "$res" "$off" > "$uscita" 2>/dev/null
    rm -f "$off" "$res" 2>/dev/null
}

risolvihashversione() {
    local v="$1" h storico="$temporanei/risolto-singolo.txt" tmp="$temporanei/catalogo-risolto.tmp"
    versionevalida "$v" || {
        print -r -- "version-hidden"
        return 1
    }
    h="$(hashversione "$v")"
    if hashvalido "$h"; then
        print -r -- "$h"
        return 0
    fi
    if curlhttps -fsSL --connect-timeout 4 --max-time 12 --retry 1 "$resolverurl" -o "$storico" 2>/dev/null; then
        h="$(grep '^New Studio version-' "$storico" 2>/dev/null | grep -F "git hash: $v " | sed -E 's/^New Studio (version-[^ ]+) .*/\1/' | head -n 1)"
        rm -f "$storico" 2>/dev/null
        if hashvalido "$h"; then
            awk -F '\t' -v v="$v" -v h="$h" 'BEGIN{OFS="\t"} {if($2==v)$3=h; print}' "$catalogo" > "$tmp" 2>/dev/null
            mv "$tmp" "$catalogo" 2>/dev/null
            normalizzacatalogo
            print -r -- "$h"
            return 0
        fi
    fi
    rm -f "$storico" "$tmp" 2>/dev/null
    print -r -- "version-hidden"
    return 1
}

aggiornacatalogo() {
    local storico="$temporanei/storico.txt"
    local risolto="$temporanei/risolto.txt"
    local ufficiale="$temporanei/ufficiale.txt"
    local unito="$temporanei/unito.txt"
    local prima dopo corrente resolverok=0
    prima="$(wc -l < "$catalogo" 2>/dev/null | tr -d ' ')"
    [[ "$prima" == <-> ]] || prima=0
    sezione "Aggiornamento catalogo"
    spinneravvia "Scarico la cronologia ufficiale Roblox"
    if ! curlhttps -fsSL --connect-timeout 6 --max-time 25 --retry 2 --retry-delay 1 "$storicourl" -o "$storico" 2>/dev/null; then
        spinnerferma
        errore "Cronologia ufficiale non raggiungibile. Controlla la connessione e riprova."
        return 1
    fi
    spinnerferma
    spinneravvia "Risolvo gli identificativi nascosti"
    if curlhttps -fsSL --connect-timeout 5 --max-time 18 --retry 1 "$resolverurl" -o "$risolto" 2>/dev/null; then
        resolverok=1
    else
        : > "$risolto"
    fi
    fondistorici "$storico" "$risolto" "$ufficiale"
    cat "$catalogo" "$ufficiale" 2>/dev/null > "$unito"
    mv "$unito" "$catalogo" 2>/dev/null
    normalizzacatalogo
    corrente="$(grep '^New Studio ' "$storico" 2>/dev/null | tail -n 1 | sed -E 's/.*git hash: ([0-9.]+).*/\1/')"
    [[ "$corrente" == 0.* ]] && printf '%s\t%s\n' "$corrente" "$(date +%s)" > "$correnteweb"
    rm -f "$storico" "$risolto" "$ufficiale" 2>/dev/null
    spinnerferma
    dopo="$(wc -l < "$catalogo" 2>/dev/null | tr -d ' ')"
    [[ "$dopo" == <-> ]] || dopo=0
    ok "Catalogo aggiornato: $dopo versioni note, $(( dopo - prima )) nuove."
    [[ -n "$corrente" ]] && campo "Versione live" "$corrente"
    if (( resolverok == 1 )); then
        campo "Hash nascosti" "risoluzione attiva"
    else
        campo "Hash nascosti" "risoluzione non raggiungibile"
    fi
    return 0
}

catalogopronto() {
    if [[ ! -s "$catalogo" ]]; then
        aggiornacatalogo || return 1
    fi
    return 0
}

sincronizzaufficiale() {
    local storico="$temporanei/storico.intel.txt"
    local risolto="$temporanei/risolto.intel.txt"
    local ufficiale="$temporanei/ufficiale.intel.txt"
    local unito="$temporanei/unito.intel.txt"
    local corrente
    if ! curlhttps -fsSL --connect-timeout 5 --max-time 20 --retry 1 "$storicourl" -o "$storico" 2>/dev/null; then
        rm -f "$storico" "$risolto" "$ufficiale" "$unito" 2>/dev/null
        return 1
    fi
    if ! curlhttps -fsSL --connect-timeout 5 --max-time 15 --retry 1 "$resolverurl" -o "$risolto" 2>/dev/null; then
        : > "$risolto"
    fi
    fondistorici "$storico" "$risolto" "$ufficiale"
    cat "$catalogo" "$ufficiale" 2>/dev/null > "$unito"
    mv "$unito" "$catalogo" 2>/dev/null
    normalizzacatalogo
    corrente="$(grep '^New Studio ' "$storico" 2>/dev/null | tail -n 1 | sed -E 's/.*git hash: ([0-9.]+).*/\1/')"
    if [[ "$corrente" == 0.* ]]; then
        printf '%s\t%s\n' "$corrente" "$(date +%s)" > "$correnteweb"
    fi
    rm -f "$storico" "$risolto" "$ufficiale" "$unito" 2>/dev/null
    return 0
}

hashversione() {
    local h
    versionevalida "$1" || return 1
    h="$(awk -F '\t' -v v="$1" '$2==v {print $3; exit}' "$catalogo" 2>/dev/null)"
    if hashvalido "$h" || [[ "$h" == "version-hidden" ]]; then
        print -r -- "$h"
        return 0
    fi
    return 1
}

dataversione() {
    awk -F '\t' -v v="$1" '$2==v {print $1; exit}' "$catalogo" 2>/dev/null
}

epochdata() {
    local d="$1" e=""
    [[ "$d" == <->-<->-<-> ]] || {
        print -r -- ""
        return 1
    }
    e="$(date -j -f '%Y-%m-%d' "$d" '+%s' 2>/dev/null)"
    if [[ "$e" != <-> ]]; then
        e="$(date -d "$d" '+%s' 2>/dev/null)"
    fi
    [[ "$e" == <-> ]] && print -r -- "$e" || print -r -- ""
}

datareferenza() {
    local v d
    v="$(versionecorrente)"
    if [[ -n "$v" ]]; then
        d="$(dataversione "$v")"
    fi
    if [[ -z "$d" ]]; then
        d="$(awk -F '\t' 'NF>=2 {print $1; exit}' "$catalogo" 2>/dev/null)"
    fi
    [[ -n "$d" ]] || d="$(date '+%Y-%m-%d')"
    print -r -- "$d"
}

fasciaversione() {
    local v="$1" d="$2" riferimento er ed giorni corrente
    [[ -n "$d" ]] || d="$(dataversione "$v")"

    corrente="$CORRENTE"
    [[ -n "$corrente" ]] || corrente="$(versionecorrente)"

    if [[ -n "$corrente" && "$v" == "$corrente" ]]; then
        print -r -- "CORRENTE"
        return 0
    fi

    riferimento="$(datareferenza)"
    er="$(epochdata "$riferimento")"
    ed="$(epochdata "$d")"

    if [[ "$er" != <-> || "$ed" != <-> ]]; then
        print -r -- "STORICA"
        return 0
    fi

    giorni=$(( (er - ed) / 86400 ))
    (( giorni < 0 )) && giorni=0

    if (( giorni <= 30 )); then
        print -r -- "RECENTE"
    elif (( giorni <= 90 )); then
        print -r -- "PRECEDENTE"
    elif (( giorni <= 180 )); then
        print -r -- "VECCHIA"
    else
        print -r -- "STORICA"
    fi
}

cdncache() {
    local h="$1" riga s t now
    now="$(date +%s)"
    riga="$(awk -F '\t' -v h="$h" '$1==h {print $2 "|" $3; exit}' "$webcache" 2>/dev/null)"
    [[ -n "$riga" ]] || return 1
    s="${riga%%|*}"
    t="${riga##*|}"
    [[ -n "$s" ]] || return 1
    [[ "$t" == <-> ]] || return 1
    (( now - t < 43200 )) || return 1
    print -r -- "$s"
    return 0
}

cdnsalva() {
    local h="$1" r="$2" tmp="$temporanei/webcache.tmp"
    hashvalido "$h" || return 1
    awk -F '\t' -v h="$h" '$1!=h {print}' "$webcache" > "$tmp" 2>/dev/null
    printf '%s\t%s\t%s\n' "$h" "$r" "$(date +%s)" >> "$tmp"
    mv "$tmp" "$webcache" 2>/dev/null
}

cdnverifica() {
    local h="$1" codice
    if [[ -z "$h" || "$h" == "version-hidden" ]]; then
        print -r -- "hash nascosto"
        return 0
    fi
    if ! hashvalido "$h"; then
        print -r -- "hash non valido"
        return 0
    fi
    codice="$(curlhttps -sIL --connect-timeout 4 --max-time 8 -o /dev/null -w '%{http_code}' "$cdnbase/$h-RobloxStudioApp.zip" 2>/dev/null)"
    case "$codice" in
        (200|206|301|302|307|308) print -r -- "disponibile" ;;
        (404|410) print -r -- "non disponibile" ;;
        (000|"") print -r -- "rete incerta" ;;
        (*) print -r -- "non verificata" ;;
    esac
}

cdnlotto() {
    local -a hash pendenti
    local h esito fatti=0 totale gruppo=0 massimo=6
    hash=("$@")
    totale=${#hash}
    CDNRIS=()
    mkdir -p "$temporanei/cdn" 2>/dev/null
    rm -f "$temporanei/cdn/"*(N) 2>/dev/null
    pendenti=()
    for h in "${hash[@]}"; do
        if [[ -z "$h" || "$h" == "version-hidden" ]]; then
            CDNRIS+=("hash nascosto")
            (( fatti++ ))
            continue
        fi
        if ! hashvalido "$h"; then
            CDNRIS+=("hash non valido")
            (( fatti++ ))
            continue
        fi
        esito="$(cdncache "$h" 2>/dev/null)"
        if [[ -n "$esito" ]]; then
            CDNRIS+=("$esito")
            (( fatti++ ))
        else
            CDNRIS+=("")
            pendenti+=("$h")
        fi
    done
    if (( ${#pendenti} == 0 )); then
        return 0
    fi
    [[ -t 1 ]] && printf '\033[?25l'
    for h in "${pendenti[@]}"; do
        { cdnverifica "$h" > "$temporanei/cdn/$h" 2>/dev/null ; } &
        (( gruppo++ ))
        if (( gruppo >= massimo )); then
            wait
            gruppo=0
            fatti=$(( fatti + massimo ))
            (( fatti > totale )) && fatti=$totale
            [[ -t 1 ]] && barra $(( fatti * 100 / totale )) "controllo CDN $fatti/$totale"
        fi
    done
    wait
    [[ -t 1 ]] && barra 100 "controllo CDN $totale/$totale"
    local i
    for (( i=1; i<=${#hash}; i++ )); do
        if [[ -z "${CDNRIS[i]}" ]]; then
            esito="$(cat "$temporanei/cdn/${hash[i]}" 2>/dev/null)"
            [[ -n "$esito" ]] || esito="non disponibile"
            CDNRIS[$i]="$esito"
            cdnsalva "${hash[i]}" "$esito"
        fi
    done
    rm -f "$temporanei/cdn/"*(N) 2>/dev/null
    spinnerferma
}

elencoapp() {
    local a
    local -a lista
    APPS=()
    lista=("${(@f)$(find "$versioni" -mindepth 2 -maxdepth 2 -type d -name '*.app' -print 2>/dev/null | sort)}")
    for a in "${lista[@]}"; do
        [[ -n "$a" && -d "$a" ]] && APPS+=("$a")
    done
}

datiapp() {
    local app="$1" cart nome
    cart="${app:h:t}"
    DATA="${cart%% *}"
    VERS="${cart#* }"
    if [[ "$DATA" != <->-<->-<-> ]]; then
        DATA="0000-00-00"
    fi
    if ! versionevalida "$VERS"; then
        nome="${app:t:r}"
        VERS="${nome##* }"
    fi
    versionevalida "$VERS" || VERS="sconosciuta"
}

trovaapp() {
    local v="$1"
    versionevalida "$v" || return 1
    find "$versioni" -mindepth 2 -maxdepth 2 -type d -name "RobloxStudio $v.app" -print -quit 2>/dev/null
}

ebloccata() {
    ls -ldO "$1" 2>/dev/null | grep -qw "uchg"
}

segnablocco() {
    local d="$1" v="$2" s="$3" tmp="$temporanei/blocco.tmp"
    awk -F '\t' -v v="$v" '$2!=v {print}' "$blocco" > "$tmp" 2>/dev/null
    printf '%s\t%s\t%s\n' "$d" "$v" "$s" >> "$tmp"
    mv "$tmp" "$blocco" 2>/dev/null
}

bloccaapp() {
    local app="$1" exe
    datiapp "$app"
    chflags uchg "$app" 2>/dev/null
    [[ -e "$app/Contents" ]] && chflags uchg "$app/Contents" 2>/dev/null
    [[ -e "$app/Contents/Info.plist" ]] && chflags uchg "$app/Contents/Info.plist" 2>/dev/null
    [[ -e "$app/Contents/MacOS" ]] && chflags uchg "$app/Contents/MacOS" 2>/dev/null
    for exe in "$app/Contents/MacOS/"*(N.); do
        chflags uchg "$exe" 2>/dev/null
    done
    segnablocco "$DATA" "$VERS" "bloccata"
}

sbloccaapp() {
    local app="$1" exe
    datiapp "$app"
    chflags nouchg "$app" 2>/dev/null
    [[ -e "$app/Contents" ]] && chflags nouchg "$app/Contents" 2>/dev/null
    [[ -e "$app/Contents/Info.plist" ]] && chflags nouchg "$app/Contents/Info.plist" 2>/dev/null
    [[ -e "$app/Contents/MacOS" ]] && chflags nouchg "$app/Contents/MacOS" 2>/dev/null
    for exe in "$app/Contents/MacOS/"*(N.); do
        chflags nouchg "$exe" 2>/dev/null
    done
    segnablocco "$DATA" "$VERS" "libera"
}

salvacompat() {
    local d="$1" v="$2" r="$3" minimo="$4" sis="$5" ar="$6" det="$7"
    local tmp="$temporanei/compat.tmp"
    awk -F '\t' -v v="$v" '$2!=v {print}' "$compatibilita" > "$tmp" 2>/dev/null
    printf '%s\t%s\t%s\t%s\t%s\t%s\t%s\n' "$d" "$v" "$r" "$minimo" "$sis" "$ar" "$det" >> "$tmp"
    sort -t $'\t' -k1,1r -k2,2r "$tmp" > "$compatibilita" 2>/dev/null
    rm -f "$tmp"
}

compatver() {
    local v="$1" riga r sis ar resto
    riga="$(awk -F '\t' -v v="$v" '$2==v {print $3 "|" $5 "|" $6; exit}' "$compatibilita" 2>/dev/null)"
    if [[ -z "$riga" ]]; then
        print -r -- "non verificata"
        return 0
    fi
    r="${riga%%|*}"
    resto="${riga#*|}"
    sis="${resto%%|*}"
    ar="${resto##*|}"
    if [[ -n "$SISTEMA" && "$sis" != "$SISTEMA" ]] || [[ -n "$ARCHMAC" && "$ar" != "$ARCHMAC" ]]; then
        print -r -- "non verificata"
        return 0
    fi
    print -r -- "$r"
}

minimover() {
    local v="$1" m
    m="$(awk -F '\t' -v v="$v" '$2==v {print $4; exit}' "$compatibilita" 2>/dev/null)"
    [[ -n "$m" ]] || m="-"
    print -r -- "$m"
}

compatapp() {
    local app="$1" dettagli="${2:-no}"
    local plist exe eseguibile minimo arches risultato dettaglio
    datiapp "$app"
    plist="$app/Contents/Info.plist"
    exe=""
    minimo=""
    risultato="sconosciuta"
    dettaglio="requisito minimo non rilevato"
    if [[ -f "$plist" ]]; then
        eseguibile="$(/usr/libexec/PlistBuddy -c 'Print :CFBundleExecutable' "$plist" 2>/dev/null)"
        [[ -n "$eseguibile" && -f "$app/Contents/MacOS/$eseguibile" ]] && exe="$app/Contents/MacOS/$eseguibile"
        minimo="$(/usr/libexec/PlistBuddy -c "Print :LSMinimumSystemVersionByArchitecture:$ARCHMAC" "$plist" 2>/dev/null)"
        [[ -n "$minimo" ]] || minimo="$(/usr/libexec/PlistBuddy -c 'Print :LSMinimumSystemVersion' "$plist" 2>/dev/null)"
        [[ -n "$minimo" ]] || minimo="$(/usr/libexec/PlistBuddy -c 'Print :MinimumSystemVersion' "$plist" 2>/dev/null)"
    fi
    [[ -n "$exe" ]] || exe="$(find "$app/Contents/MacOS" -maxdepth 1 -type f -print -quit 2>/dev/null)"
    if [[ -z "$exe" ]]; then
        [[ -n "$minimo" ]] || minimo="sconosciuto"
        salvacompat "$DATA" "$VERS" "non avviabile" "$minimo" "$SISTEMA" "$ARCHMAC" "eseguibile non trovato"
        return 2
    fi
    arches="$(/usr/bin/lipo -archs "$exe" 2>/dev/null)"
    [[ -n "$arches" ]] || arches="$(file "$exe" 2>/dev/null)"
    if [[ "$ARCHMAC" == "x86_64" ]]; then
        if ! print -r -- "$arches" | grep -q "x86_64"; then
            risultato="cpu"
            dettaglio="eseguibile senza codice x86_64"
        fi
    elif [[ "$ARCHMAC" == "arm64" ]]; then
        if print -r -- "$arches" | grep -q "arm64"; then
            :
        elif print -r -- "$arches" | grep -q "x86_64"; then
            if /usr/bin/pgrep oahd >/dev/null 2>&1 || [[ -e "/Library/Apple/usr/libexec/oah" ]]; then
                risultato="compatibile"
                dettaglio="x86_64 tramite Rosetta"
            else
                risultato="cpu"
                dettaglio="richiede Rosetta non installato"
            fi
        else
            risultato="cpu"
            dettaglio="architettura non compatibile"
        fi
    fi
    if [[ -z "$minimo" ]]; then
        minimo="$(otool -l "$exe" 2>/dev/null | awk '
            /cmd LC_BUILD_VERSION/ {b=1; m=0; next}
            b && /^[ \t]*minos[ \t]+/ {print $2; exit}
            /cmd LC_VERSION_MIN_MACOSX/ {m=1; b=0; next}
            m && /^[ \t]*version[ \t]+/ {print $2; exit}
        ')"
    fi
    [[ -n "$minimo" ]] || minimo="sconosciuto"
    if [[ "$risultato" != "cpu" ]]; then
        if [[ "$minimo" != "sconosciuto" && -n "$SISTEMA" ]]; then
            if versioneminore "$SISTEMA" "$minimo"; then
                risultato="macos vecchio"
                dettaglio="richiede macOS $minimo o successivo"
            else
                risultato="compatibile"
                dettaglio="macOS $SISTEMA soddisfa il requisito minimo $minimo"
            fi
        elif [[ "$risultato" != "compatibile" ]]; then
            risultato="sconosciuta"
            dettaglio="requisito minimo non rilevato"
        fi
    fi
    salvacompat "$DATA" "$VERS" "$risultato" "$minimo" "$SISTEMA" "$ARCHMAC" "$dettaglio"
    if [[ "$dettagli" == "si" ]]; then
        sezione "Compatibilita con questo Mac"
        campo "Versione" "$VERS"
        campo "Data build" "$(datavisibile "$DATA")"
        campo "macOS attuale" "$SISTEMA"
        campo "macOS minimo" "$minimo"
        campo "Processore" "$ARCHMAC"
        campo "Esito" "$risultato"
        info "$dettaglio"
    fi
    [[ "$risultato" == "compatibile" ]]
}

compatassicura() {
    local app="$1" v="$2"
    if [[ "$(compatver "$v")" == "non verificata" ]]; then
        compatapp "$app" "no" >/dev/null 2>&1
    fi
}

salvastato() {
    local d="$1" v="$2" r="$3" det="$4" tmp="$temporanei/stato.tmp"
    awk -F '\t' -v v="$v" '$2!=v {print}' "$stato" > "$tmp" 2>/dev/null
    printf '%s\t%s\t%s\t%s\t%s\n' "$d" "$v" "$r" "$det" "$(date '+%Y-%m-%d %H:%M')" >> "$tmp"
    sort -t $'\t' -k1,1r -k2,2r "$tmp" > "$stato" 2>/dev/null
    rm -f "$tmp"
}

statover() {
    local v="$1" s
    s="$(awk -F '\t' -v v="$v" '$2==v {print $3; exit}' "$stato" 2>/dev/null)"
    [[ -n "$s" ]] || s="noncontrollata"
    print -r -- "$s"
}

testover() {
    local v="$1" t
    t="$(awk -F '\t' -v v="$v" '$2==v {print $5; exit}' "$stato" 2>/dev/null)"
    [[ -n "$t" ]] || t="-"
    print -r -- "$t"
}

epochatest() {
    local t="$1" e=""
    [[ -n "$t" && "$t" != "-" ]] || {
        print -r -- ""
        return 1
    }
    e="$(date -j -f '%Y-%m-%d %H:%M' "$t" '+%s' 2>/dev/null)"
    if [[ "$e" != <-> ]]; then
        e="$(date -d "$t" '+%s' 2>/dev/null)"
    fi
    [[ "$e" == <-> ]] && print -r -- "$e" || print -r -- ""
}

testfresco() {
    local v="$1" limite="${2:-$VERIFICA_TTL}" t e now
    t="$(testover "$v")"
    [[ "$t" != "-" ]] || return 1
    e="$(epochatest "$t")"
    [[ "$e" == <-> ]] || return 1
    now="$(date +%s)"
    (( now >= e && now - e <= limite ))
}

LIMOBSOLETA=""
LIMSUPPORTATA=""
CORRENTE=""

calcolalimiti() {
    local d td tv ts tr tt
    LIMOBSOLETA="$(awk -F '	' '$3=="obsoleta" {print $1}' "$stato" 2>/dev/null | sort -r | head -n 1)"
    LIMSUPPORTATA=""
    while IFS=$'	' read -r td tv ts tr tt; do
        [[ "$ts" == "supportata" ]] || continue
        testfresco "$tv" "$VERIFICA_TTL" || continue
        if [[ -z "$LIMSUPPORTATA" || "$td" < "$LIMSUPPORTATA" ]]; then
            LIMSUPPORTATA="$td"
        fi
    done < "$stato"
    CORRENTE="$(awk -F '	' 'NR==1 {print $1}' "$correnteweb" 2>/dev/null)"
    [[ "$CORRENTE" == 0.* ]] || CORRENTE=""
    if [[ -n "$CORRENTE" ]]; then
        d="$(dataversione "$CORRENTE")"
        if [[ -n "$d" ]]; then
            if [[ -z "$LIMSUPPORTATA" || "$d" < "$LIMSUPPORTATA" ]]; then
                LIMSUPPORTATA="$d"
            fi
        fi
    fi
    if [[ -n "$LIMOBSOLETA" && -n "$LIMSUPPORTATA" ]]; then
        if [[ "$LIMOBSOLETA" > "$LIMSUPPORTATA" || "$LIMOBSOLETA" == "$LIMSUPPORTATA" ]]; then
            LIMSUPPORTATA=""
        fi
    fi
}

statoroblox() {
    local v="$1" d="$2" s
    s="$(statover "$v")"
    case "$s" in
        (supportata)
            if testfresco "$v" "$VERIFICA_TTL"; then
                print -r -- "valida"
            else
                print -r -- "da verificare"
            fi
            return 0
            ;;
        (obsoleta)
            if testfresco "$v" "$OOD_TTL"; then
                print -r -- "OUT OF DATE"
            else
                print -r -- "da verificare"
            fi
            return 0
            ;;
        (nonavviabile) print -r -- "non avviabile"; return 0 ;;
        (incerta)
            if testfresco "$v" 10800; then
                print -r -- "incerta"
            else
                print -r -- "da verificare"
            fi
            return 0
            ;;
    esac
    print -r -- "da verificare"
}

verdetto() {
    local v="$1" d="$2" c s
    c="$(compatver "$v")"
    s="$(statoroblox "$v" "$d")"
    VERDETTO=""
    MOTIVO=""

    case "$c" in
        ("macos vecchio")
            VERDETTO="non avviabile"
            MOTIVO="questa build richiede macOS $(minimover "$v") mentre il Mac ha $SISTEMA"
            return 1
            ;;
        (cpu)
            VERDETTO="non avviabile"
            MOTIVO="l'eseguibile non e compatibile con il processore $ARCHMAC"
            return 1
            ;;
        ("non avviabile")
            VERDETTO="non avviabile"
            MOTIVO="il pacchetto e incompleto o danneggiato"
            return 1
            ;;
    esac

    case "$s" in
        (valida)
            VERDETTO="avviabile"
            MOTIVO="verificata sul Mac senza avvisi di aggiornamento obbligatorio"
            return 0
            ;;
        ("OUT OF DATE")
            VERDETTO="out of date"
            MOTIVO="Roblox ha imposto l'aggiornamento durante la verifica"
            return 2
            ;;
        ("non avviabile")
            VERDETTO="non avviabile"
            MOTIVO="la build non supera i controlli di avvio"
            return 1
            ;;
        (incerta)
            VERDETTO="incerta"
            MOTIVO="la verifica non ha prodotto un risultato affidabile"
            return 3
            ;;
    esac

    VERDETTO="da verificare"
    MOTIVO="serve una verifica reale per sapere se Roblox la accetta ancora"
    return 3
}


INT_ESITO=""
INT_CONF=""
INT_FONTE=""

intelligenzaversione() {
    local v="$1" d="$2" sr
    sr="$(statoroblox "$v" "$d")"
    INT_ESITO="DA TESTARE"
    INT_CONF="0%"
    INT_FONTE="nessun test recente"
    case "$sr" in
        (valida)
            INT_ESITO="AVVIABILE"
            INT_CONF="100%"
            INT_FONTE="test reale recente"
            ;;
        ("OUT OF DATE")
            INT_ESITO="OUT OF DATE"
            INT_CONF="100%"
            INT_FONTE="test reale recente"
            ;;
        ("non avviabile")
            INT_ESITO="NON AVVIABILE"
            INT_CONF="100%"
            INT_FONTE="compatibilita Mac"
            ;;
        (incerta)
            INT_ESITO="INCERTA"
            INT_CONF="30%"
            INT_FONTE="test incompleto"
            ;;
    esac
}

scegliriga() {
    local pagina=1 perpagina totale pagine i inizio fine inp corrente
    local -a campi
    totale=${#SC_RIGHE}
    SC_SEL=0
    if (( totale == 0 )); then
        avviso "Nessun elemento da mostrare."
        return 1
    fi
    while true; do
        titolo
        sezione "$SC_TITOLO"
        perpagina=$(( RIGHE - 16 ))
        (( perpagina < 2 )) && perpagina=2
        (( perpagina > 40 )) && perpagina=40
        pagine=$(( (totale + perpagina - 1) / perpagina ))
        (( pagina > pagine )) && pagina=$pagine
        (( pagina < 1 )) && pagina=1
        inizio=$(( (pagina - 1) * perpagina + 1 ))
        fine=$(( inizio + perpagina - 1 ))
        (( fine > totale )) && fine=$totale
        tabinit "${SC_SPEC[@]}"
        tabbordo
        tabtesta
        tabbordo
        for (( i=inizio; i<=fine; i++ )); do
            corrente="${SC_RIGHE[i]}"
            campi=("${(@ps:\t:)corrente}")
            tabriga "$i" "${campi[@]}"
        done
        tabbordo
        print
        [[ -n "$SC_NOTA" ]] && nota "$SC_NOTA"
        nota "Pagina $pagina di $pagine   numero = scegli   n = avanti   p = indietro   0 = torna"
        print
        domanda "Scelta:"
        IFS= read -r inp || return 1
        case "$inp" in
            (n|N) (( pagina < pagine )) && (( pagina++ )) ;;
            (p|P) (( pagina > 1 )) && (( pagina-- )) ;;
            (0) return 1 ;;
            (<->)
                if (( inp >= 1 && inp <= totale )); then
                    SC_SEL=$inp
                    return 0
                fi
                ;;
            (*) ;;
        esac
    done
}

righeinstallate() {
    local app s c b t f
    RIGHEAPP=()
    calcolalimiti
    elencoapp
    for app in "${APPS[@]}"; do
        datiapp "$app"
        compatassicura "$app" "$VERS"
        s="$(statoroblox "$VERS" "$DATA")"
        c="$(compatver "$VERS")"
        t="$(testover "$VERS")"
        f="$(fasciaversione "$VERS" "$DATA")"
        if ebloccata "$app"; then
            b="bloccata"
        else
            b="libera"
        fi
        RIGHEAPP+=("$(datavisibile "$DATA")"$'\t'"$VERS"$'\t'"$f"$'\t'"$s"$'\t'"$c"$'\t'"$b"$'\t'"$t")
    done
}

specinstallate() {
    SC_SPEC=("N|3|3|0" "Data|10|10|1" "Versione|11|16|0" "Eta|8|10|0" "Roblox|11|13|0" "Mac|8|14|2" "Blocco|7|8|3" "Verifica|10|16|4")
}

mostrainstallate() {
    local i corrente
    local -a campi
    righeinstallate
    if (( ${#RIGHEAPP} == 0 )); then
        sezione "Versioni installate"
        avviso "Nessuna versione presente nella cartella Versioni."
        nota "Usa Versioni per cercare e scaricare una build."
        return 1
    fi
    sezione "Versioni installate"
    specinstallate
    tabinit "${SC_SPEC[@]}"
    tabbordo
    tabtesta
    tabbordo
    for (( i=1; i<=${#RIGHEAPP}; i++ )); do
        corrente="${RIGHEAPP[i]}"
        campi=("${(@ps:\t:)corrente}")
        tabriga "$i" "${campi[@]}"
    done
    tabbordo
    print
    nota "Eta: RECENTE entro 30 giorni, PRECEDENTE 31-90, VECCHIA 91-180, STORICA oltre 180."
    nota "Lo stato Roblox e considerato valido solo quando deriva da una verifica recente."
    return 0
}

selezionaapp() {
    righeinstallate
    if (( ${#RIGHEAPP} == 0 )); then
        avviso "Nessuna versione presente nella cartella Versioni."
        return 1
    fi
    SC_TITOLO="$1"
    specinstallate
    SC_RIGHE=("${RIGHEAPP[@]}")
    SC_NOTA="Eta separa le build per periodo. L'asterisco indica uno stato Roblox dedotto."
    scegliriga || return 1
    APPSCELTA="${APPS[SC_SEL]}"
    return 0
}

migliorescelta() {
    local app s c migliore="" migliored=""
    righeinstallate
    for app in "${APPS[@]}"; do
        datiapp "$app"
        c="$(compatver "$VERS")"
        [[ "$c" == "compatibile" || "$c" == "sconosciuta" ]] || continue
        s="$(statoroblox "$VERS" "$DATA")"
        case "$s" in
            (corrente|valida|"valida*")
                if [[ -z "$migliored" || "$DATA" > "$migliored" ]]; then
                    migliore="$app"
                    migliored="$DATA"
                fi
                ;;
        esac
    done
    print -r -- "$migliore"
}

dimensioneremota() {
    local url="$1" n
    n="$(curlhttps -sIL --connect-timeout 5 --max-time 12 "$url" 2>/dev/null | awk '{k=tolower($1)} k=="content-length:" {n=$2} END{gsub(/\r/,"",n); print n+0}')"
    [[ "$n" == <-> ]] || n=0
    print -r -- "$n"
}

scaricafile() {
    local url="$1" dest="$2" tot="$3"
    local pid cur pct vel t0 now dt eta rc testo
    rm -f "$dest" 2>/dev/null
    curlhttps -fsSL --connect-timeout 8 --max-time 5400 --retry 2 --retry-delay 2 -o "$dest" "$url" 2>/dev/null &
    pid=$!
    t0="$(date +%s)"
    [[ -t 1 ]] && printf '\033[?25l'
    while kill -0 "$pid" 2>/dev/null; do
        cur="$(dimensionefile "$dest")"
        now="$(date +%s)"
        dt=$(( now - t0 ))
        (( dt < 1 )) && dt=1
        vel=$(( cur / dt ))
        if [[ -t 1 ]]; then
            if (( tot > 0 )); then
                pct=$(( cur * 100 / tot ))
                (( pct > 100 )) && pct=100
                if (( vel > 0 )); then
                    eta=$(( (tot - cur) / vel ))
                    (( eta < 0 )) && eta=0
                    testo="$(megabyte $cur)/$(megabyte $tot) MB  $(megabyte $vel) MB/s  $(durata $eta)"
                else
                    testo="$(megabyte $cur)/$(megabyte $tot) MB"
                fi
                barra "$pct" "$testo"
            else
                barra $(( (dt * 7) % 100 )) "$(megabyte $cur) MB scaricati"
            fi
        fi
        sleep 0.25
    done
    wait "$pid"
    rc=$?
    spinnerferma
    return $rc
}

zipvalido() {
    local f="$1" n voce
    [[ -f "$f" && ! -L "$f" ]] || return 1
    n="$(dimensionefile "$f")"
    [[ "$n" == <-> ]] || return 1
    (( n >= 1048576 && n <= 4294967296 )) || return 1
    /usr/bin/unzip -tqq "$f" >/dev/null 2>&1 || return 1
    while IFS= read -r voce; do
        [[ -n "$voce" ]] || continue
        case "$voce" in
            (/*|..|../*|*/../*|*/..) return 1 ;;
        esac
    done < <(/usr/bin/unzip -Z1 "$f" 2>/dev/null)
    return 0
}

verificaappufficiale() {
    local app="$1" attesa="${2:-}" info team ident breve build requisito
    [[ -d "$app" && -f "$app/Contents/Info.plist" ]] || return 1
    requisito='anchor apple generic and certificate 1[field.1.2.840.113635.100.6.2.6] exists and certificate leaf[field.1.2.840.113635.100.6.1.13] exists and certificate leaf[subject.OU] = "2CFABCH843"'
    /usr/bin/codesign --verify --deep --strict -R="$requisito" "$app" >/dev/null 2>&1 || return 1
    info="$(/usr/bin/codesign -dvvv "$app" 2>&1)"
    team="$(print -r -- "$info" | sed -n 's/^TeamIdentifier=//p' | head -n 1)"
    ident="$(print -r -- "$info" | sed -n 's/^Identifier=//p' | head -n 1)"
    [[ "$team" == "2CFABCH843" ]] || return 1
    [[ "${(L)ident}" == "com.roblox.robloxstudio" ]] || return 1
    print -r -- "$info" | grep -Fqi 'Developer ID Application: Roblox Corporation (2CFABCH843)' || return 1
    if [[ -n "$attesa" ]]; then
        versionevalida "$attesa" || return 1
        breve="$(/usr/libexec/PlistBuddy -c 'Print :CFBundleShortVersionString' "$app/Contents/Info.plist" 2>/dev/null)"
        build="$(/usr/libexec/PlistBuddy -c 'Print :CFBundleVersion' "$app/Contents/Info.plist" 2>/dev/null)"
        [[ "$breve" == "$attesa" || "$build" == "$attesa" ]] || return 1
    fi
    return 0
}

preparaappmac() {
    local app="$1" attesa="${2:-}"
    [[ -d "$app" ]] || return 1
    verificaappufficiale "$app" "$attesa" || return 2
    xattr -dr com.apple.quarantine "$app" 2>/dev/null || true
    return 0
}

trovaarchivio() {
    local v="$1" f
    for f in "$archivio"/*(N) "$HOME/Downloads"/*(N); do
        [[ -e "$f" ]] || continue
        case "${f:t}" in
            (*"$v"*.zip|*"$v"*.dmg|*"$v"*.app)
                print -r -- "$f"
                return 0
                ;;
        esac
    done
    return 1
}

preparaarchivio() {
    local fonte="$1" destinazione="$2" ext="${fonte:e:l}" mountpoint sorgente
    rm -rf "$destinazione" 2>/dev/null
    mkdir -p "$destinazione" 2>/dev/null

    if [[ -d "$fonte" && "${fonte:e:l}" == "app" ]]; then
        ditto "$fonte" "$destinazione/${fonte:t}" >/dev/null 2>&1 || return 1
        print -r -- "$destinazione/${fonte:t}"
        return 0
    fi

    case "$ext" in
        (zip)
            zipvalido "$fonte" || return 1
            ditto -x -k "$fonte" "$destinazione" >/dev/null 2>&1 || return 1
            ;;
        (dmg)
            mountpoint="$temporanei/mount.$$"
            mkdir -p "$mountpoint" 2>/dev/null
            if ! hdiutil attach "$fonte" -nobrowse -readonly -mountpoint "$mountpoint" >/dev/null 2>&1; then
                rm -rf "$mountpoint" 2>/dev/null
                return 1
            fi
            sorgente="$(find "$mountpoint" -maxdepth 4 -type d \( -name 'RobloxStudio.app' -o -name 'Roblox Studio.app' -o -name 'RobloxStudio*.app' \) -print -quit 2>/dev/null)"
            if [[ -n "$sorgente" ]]; then
                ditto "$sorgente" "$destinazione/${sorgente:t}" >/dev/null 2>&1
            fi
            hdiutil detach "$mountpoint" -force >/dev/null 2>&1
            rm -rf "$mountpoint" 2>/dev/null
            [[ -n "$sorgente" ]] || return 1
            ;;
        (*)
            return 1
            ;;
    esac

    sorgente="$(find "$destinazione" -maxdepth 6 -type d \( -name 'RobloxStudio.app' -o -name 'Roblox Studio.app' -o -name 'RobloxStudio*.app' \) -print -quit 2>/dev/null)"
    [[ -n "$sorgente" ]] || return 1
    print -r -- "$sorgente"
    return 0
}

installadaarchivio() {
    local v="$1" riga d h dummy fonte risposta destinazione app prova sorgente rc sostituisci=0
    versionevalida "$v" || { errore "Versione non valida."; return 1; }
    catalogopronto || return 1
    riga="$(awk -F '\t' -v v="$v" '$2==v {print; exit}' "$catalogo" 2>/dev/null)"
    [[ -n "$riga" ]] || {
        errore "Versione non presente nel catalogo."
        return 1
    }
    IFS=$'\t' read -r d dummy h <<< "$riga"
    fonte="$(trovaarchivio "$v")"
    if [[ -z "$fonte" ]]; then
        sezione "Archivio locale"
        campo "Versione" "$v"
        campo "Data build" "$(datavisibile "$d")"
        avviso "Il pacchetto ufficiale non e disponibile e non ho trovato una copia locale."
        nota "Metti una copia che possiedi in:"
        campo "Archivio" "$archivio"
        nota "Formati supportati: .zip, .dmg e .app."
        nota "Il nome del file deve contenere il numero di versione, per esempio $v."
        return 1
    fi

    destinazione="$versioni/$d $v"
    app="$destinazione/RobloxStudio $v.app"

    if [[ -e "$destinazione" ]]; then
        avviso "Questa versione e gia presente."
        domanda "Sostituirla con la copia in Archivio [s/N]:"
        IFS= read -r risposta || return 0
        case "${(L)risposta}" in
            (s|si|y|yes) sostituisci=1 ;;
            (*) return 0 ;;
        esac
    fi

    prova="$temporanei/import-$v"
    spinneravvia "Verifico la copia locale"
    sorgente="$(preparaarchivio "$fonte" "$prova")"
    rc=$?
    spinnerferma

    if (( rc != 0 )) || [[ -z "$sorgente" || ! -d "$sorgente" ]]; then
        errore "La copia locale non contiene una Roblox Studio.app utilizzabile."
        rm -rf "$prova" 2>/dev/null
        return 1
    fi
    if ! verificaappufficiale "$sorgente" "$v"; then
        errore "La copia locale non ha una firma Roblox valida o non corrisponde alla versione richiesta."
        rm -rf "$prova" 2>/dev/null
        return 1
    fi

    if (( sostituisci == 1 )); then
        [[ -d "$app" ]] && sbloccaapp "$app"
        rm -rf "$destinazione" 2>/dev/null
        [[ ! -e "$destinazione" ]] || {
            errore "Non riesco a sostituire la versione attuale."
            rm -rf "$prova" 2>/dev/null
            return 1
        }
        rimuovirecord "$v"
    fi

    mkdir -p "$destinazione" 2>/dev/null
    ditto "$sorgente" "$app" >/dev/null 2>&1
    rm -rf "$prova" 2>/dev/null

    [[ -d "$app" ]] || {
        errore "Importazione non riuscita."
        return 1
    }
    if ! preparaappmac "$app" "$v"; then
        errore "La verifica finale della firma Roblox non e riuscita."
        rm -rf "$destinazione" 2>/dev/null
        return 1
    fi

    bloccaapp "$app"
    compatapp "$app" "no" >/dev/null 2>&1 || true
    ok "Versione $v importata dall'Archivio locale."
    campo "Data build" "$(datavisibile "$d")"
    return 0
}

modalitaarchivio() {
    local app="$1"
    [[ -d "$app" ]] || return 1
    datiapp "$app"
    if ! verificaappufficiale "$app" "$VERS"; then
        errore "Avvio bloccato: firma Roblox non valida o applicazione modificata."
        return 1
    fi
    titolo
    sezione "Modalita archivio"
    campo "Versione" "$VERS"
    campo "Data build" "$(datavisibile "$DATA")"
    avviso "Avvio locale con protezione dagli aggiornamenti."
    nota "Questa modalita non modifica la build e non aggira un rifiuto OUT OF DATE dei server Roblox."
    bloccaapp "$app"
    open -na "$app" >/dev/null 2>&1
}

scaricaversione() {
    local v="$1" riga d h dummy destinazione app zip estrazione sorgente tot esito risposta sostituisci=0
    versionevalida "$v" || { errore "Versione non valida."; return 1; }
    catalogopronto || return 1
    riga="$(awk -F '	' -v v="$v" '$2==v {print; exit}' "$catalogo" 2>/dev/null)"
    if [[ -z "$riga" ]]; then
        errore "Versione non presente nel catalogo."
        return 1
    fi
    IFS=$'	' read -r d dummy h <<< "$riga"
    destinazione="$versioni/$d $v"
    app="$destinazione/RobloxStudio $v.app"
    sezione "Download versione $v"
    campo "Data build" "$(datavisibile "$d")"
    campo "Identificativo" "$h"
    if [[ -e "$destinazione" ]]; then
        print
        avviso "Questa versione e gia stata scaricata."
        campo "Versione locale" "$v"
        print
        avviso "Se continui, i file attuali verranno sostituiti dalla nuova copia."
        nota "La copia esistente resta intatta finche download ed estrazione non sono completati."
        print
        domanda "Riscaricare e sostituire [s/N]:"
        IFS= read -r risposta || return 0
        case "${(L)risposta}" in
            (s|si|y|yes) sostituisci=1 ;;
            (*)
                nota "Download annullato. I file attuali non sono stati modificati."
                return 0
                ;;
        esac
    fi
    spinneravvia "Controllo la disponibilita sul CDN Roblox"
    esito="$(cdnverifica "$h")"
    cdnsalva "$h" "$esito"
    spinnerferma
    if [[ "$esito" != "disponibile" ]]; then
        if [[ "$esito" == "rete incerta" || "$esito" == "non verificata" ]]; then
            avviso "Non posso confermare la disponibilita del pacchetto per un problema di rete."
            nota "Non lo considero rimosso. Riprova tra poco oppure usa una copia nel tuo Archivio."
        elif [[ "$esito" == "hash nascosto" ]]; then
            avviso "L'identificativo della build e ancora nascosto e non e stato possibile risolverlo."
        else
            avviso "Il pacchetto non risponde sul CDN ufficiale."
        fi
        nota "Cerco anche una copia storica nel tuo Archivio locale."
        installadaarchivio "$v"
        return $?
    fi
    zip="$temporanei/$h.zip"
    estrazione="$temporanei/$h"
    rm -rf "$estrazione" 2>/dev/null
    mkdir -p "$estrazione" 2>/dev/null
    spinneravvia "Calcolo la dimensione del pacchetto"
    tot="$(dimensioneremota "$cdnbase/$h-RobloxStudioApp.zip")"
    spinnerferma
    print
    if ! scaricafile "$cdnbase/$h-RobloxStudioApp.zip" "$zip" "$tot"; then
        errore "Download interrotto. La copia gia presente non e stata modificata."
        rm -f "$zip" 2>/dev/null
        rm -rf "$estrazione" 2>/dev/null
        return 1
    fi
    ok "Pacchetto scaricato: $(megabyte "$(dimensionefile "$zip")") MB"
    spinneravvia "Verifico ed estraggo l'applicazione"
    if ! zipvalido "$zip"; then
        spinnerferma
        errore "Il pacchetto non supera i controlli di sicurezza."
        rm -f "$zip" 2>/dev/null
        rm -rf "$estrazione" 2>/dev/null
        return 1
    fi
    if ! ditto -x -k "$zip" "$estrazione" >/dev/null 2>&1; then
        spinnerferma
        errore "Archivio danneggiato. La copia gia presente non e stata modificata."
        rm -f "$zip" 2>/dev/null
        rm -rf "$estrazione" 2>/dev/null
        return 1
    fi
    spinnerferma
    sorgente="$(find "$estrazione" -maxdepth 5 -type d \( -name 'RobloxStudio.app' -o -name 'Roblox Studio.app' \) -print -quit 2>/dev/null)"
    if [[ -z "$sorgente" ]]; then
        errore "Nell'archivio non c'e nessuna applicazione Roblox Studio."
        rm -f "$zip" 2>/dev/null
        rm -rf "$estrazione" 2>/dev/null
        return 1
    fi
    if ! verificaappufficiale "$sorgente" "$v"; then
        errore "Firma Roblox non valida o versione del pacchetto non corrispondente."
        rm -f "$zip" 2>/dev/null
        rm -rf "$estrazione" 2>/dev/null
        return 1
    fi
    if (( sostituisci == 1 )); then
        spinneravvia "Sostituisco la versione esistente"
        [[ -d "$app" ]] && sbloccaapp "$app"
        rm -rf "$destinazione" 2>/dev/null
        spinnerferma
        if [[ -e "$destinazione" ]]; then
            errore "Non riesco a sostituire la cartella attuale."
            rm -f "$zip" 2>/dev/null
            rm -rf "$estrazione" 2>/dev/null
            return 1
        fi
    fi
    mkdir -p "$destinazione" 2>/dev/null
    mv "$sorgente" "$app" 2>/dev/null
    if [[ ! -d "$app" ]]; then
        errore "Copia dell'applicazione non riuscita."
        rm -f "$zip" 2>/dev/null
        rm -rf "$estrazione" 2>/dev/null
        return 1
    fi
    rm -rf "$estrazione" 2>/dev/null
    rm -f "$zip" 2>/dev/null
    if ! preparaappmac "$app" "$v"; then
        errore "La verifica finale della firma Roblox non e riuscita."
        rm -rf "$destinazione" 2>/dev/null
        return 1
    fi
    if (( sostituisci == 1 )); then
        rimuovirecord "$v"
    fi
    spinneravvia "Blocco gli aggiornamenti automatici"
    bloccaapp "$app"
    spinnerferma
    if (( sostituisci == 1 )); then
        ok "Versione $v riscaricata e sostituita correttamente."
    else
        ok "Versione $v installata correttamente."
    fi
    compatapp "$app" "si"
    return 0
}

cercacatalogo() {
    local inp datachiesta i n d v h s cdn locale
    local -a righe hash filtrate
    catalogopronto || return 0
    sezione "Catalogo Roblox Studio"
    info "Cerca per anno, per data oppure per numero di versione."
    nota "Esempi: 2025   19/08/2025   0.712   ultima"
    print
    domanda "Ricerca:"
    IFS= read -r inp || return 0
    inp="$(print -r -- "$inp" | tr -d '[:space:]')"
    [[ -z "$inp" ]] && return 0
    if [[ "$inp" == "ultima" || "$inp" == "corrente" ]]; then
        v="$(versionecorrente)"
        if [[ -z "$v" ]]; then
            errore "Impossibile leggere la versione pubblicata."
            return 0
        fi
        inp="$v"
    fi
    if [[ "$inp" == <->-<->-<-> || "$inp" == <->/<->/<-> ]]; then
        datachiesta="$(normalizzadata "$inp")"
        righe=("${(@f)$(awk -F '\t' -v d="$datachiesta" '$1==d {print}' "$catalogo" 2>/dev/null)}")
        SC_TITOLO="Build del $(datavisibile "$datachiesta")"
    elif [[ "$inp" == <->  && ${#inp} == 4 ]]; then
        righe=("${(@f)$(awk -F '\t' -v a="$inp" 'substr($1,1,4)==a {print}' "$catalogo" 2>/dev/null)}")
        SC_TITOLO="Build dell'anno $inp"
    else
        righe=("${(@f)$(awk -F '\t' -v p="$inp" 'index($2,p)==1 {print}' "$catalogo" 2>/dev/null)}")
        SC_TITOLO="Risultati per $inp"
    fi
    filtrate=()
    for i in "${righe[@]}"; do
        [[ -n "$i" ]] && filtrate+=("$i")
    done
    righe=("${filtrate[@]}")
    if (( ${#righe} > 1 )); then
        righe=("${(@f)$(printf '%s\n' "${righe[@]}" | sort -t $'\t' -k1,1 -k2,2)}")
    fi
    if (( ${#righe} == 0 )); then
        avviso "Nessuna build corrisponde a questa ricerca."
        return 0
    fi
    if (( ${#righe} > 60 )); then
        righe=("${(@)righe[1,60]}")
        SC_TITOLO="$SC_TITOLO (prime 60)"
    fi
    hash=()
    for i in "${righe[@]}"; do
        IFS=$'\t' read -r d v h <<< "$i"
        hash+=("$h")
    done
    print
    nota "Controllo la disponibilita delle build sui server Roblox."
    cdnlotto "${hash[@]}"
    calcolalimiti
    SC_RIGHE=()
    for (( i=1; i<=${#righe}; i++ )); do
        IFS=$'\t' read -r d v h <<< "${righe[i]}"
        cdn="${CDNRIS[i]}"
        [[ -n "$cdn" ]] || cdn="non disponibile"
        if [[ -n "$(trovaapp "$v")" ]]; then
            locale="si"
        else
            locale="no"
        fi
        s="$(statoroblox "$v" "$d")"
        f="$(fasciaversione "$v" "$d")"
        SC_RIGHE+=("$(datavisibile "$d")"$'\t'"$v"$'\t'"$f"$'\t'"$cdn"$'\t'"$locale"$'\t'"$s")
    done
    SC_SPEC=("N|3|3|0" "Data|10|10|1" "Versione|11|16|0" "Eta|8|10|0" "CDN|3|15|3" "Disco|3|5|4" "Roblox|11|13|0")
    SC_NOTA="Eta = CORRENTE, RECENTE, PRECEDENTE, VECCHIA o STORICA. Non indica da sola se una build e out of date."
    scegliriga || return 0
    IFS=$'\t' read -r d v h <<< "${righe[SC_SEL]}"
    cdn="${CDNRIS[SC_SEL]}"
    titolo
    sezione "Versione $v"
    campo "Data build" "$(datavisibile "$d")"
    calcolalimiti
    campo "Eta build" "$(fasciaversione "$v" "$d")"
    campo "Disponibilita CDN" "$cdn"
    s="$(statoroblox "$v" "$d")"
    campo "Stato Roblox" "$s"
    intelligenzaversione "$v" "$d"
    campo "Valutazione" "$INT_ESITO"
    campo "Affidabilita" "$INT_CONF"
    campo "Fonte stato" "$INT_FONTE"
    print
    voce 1 "Scarica o riscarica questa versione"
    voce 2 "Verifica out of date senza conservarla"
    voce 0 "Torna indietro"
    print
    domanda "Scelta:"
    IFS= read -r inp || return 0
    case "$inp" in
        (1)
            if [[ "$cdn" != "disponibile" ]]; then
                avviso "Questa build non e disponibile sul CDN ufficiale."
                installadaarchivio "$v"
                return 0
            fi
            if [[ "$s" == "OUT OF DATE" || "$s" == "OUT OF DATE*" ]]; then
                print
                avviso "Questa build risulta rifiutata da Roblox."
                domanda "Scaricarla comunque [s/N]:"
                IFS= read -r inp || return 0
                [[ "$inp" == [sSyY] ]] || return 0
            fi
            scaricaversione "$v"
            ;;
        (2)
            verificabuild "$v"
            ;;
        (*) return 0 ;;
    esac
}

proteggiprova() {
    local app="$1" exe
    chflags uchg "$app" 2>/dev/null
    [[ -e "$app/Contents" ]] && chflags uchg "$app/Contents" 2>/dev/null
    [[ -e "$app/Contents/Info.plist" ]] && chflags uchg "$app/Contents/Info.plist" 2>/dev/null
    [[ -e "$app/Contents/MacOS" ]] && chflags uchg "$app/Contents/MacOS" 2>/dev/null
    for exe in "$app/Contents/MacOS/"*(N.); do
        chflags uchg "$exe" 2>/dev/null
    done
}

sproteggiprova() {
    local app="$1" exe
    chflags nouchg "$app" 2>/dev/null
    [[ -e "$app/Contents" ]] && chflags nouchg "$app/Contents" 2>/dev/null
    [[ -e "$app/Contents/Info.plist" ]] && chflags nouchg "$app/Contents/Info.plist" 2>/dev/null
    [[ -e "$app/Contents/MacOS" ]] && chflags nouchg "$app/Contents/MacOS" 2>/dev/null
    for exe in "$app/Contents/MacOS/"*(N.); do
        chflags nouchg "$exe" 2>/dev/null
    done
}

verificatemp() {
    local v="$1" riga d h dummy zip estrazione cartella app sorgente tot esito rc=4
    versionevalida "$v" || { errore "Versione non valida."; return 1; }
    riga="$(awk -F '\t' -v v="$v" '$2==v {print; exit}' "$catalogo" 2>/dev/null)"
    [[ -n "$riga" ]] || {
        errore "Versione non presente nel catalogo."
        return 1
    }
    IFS=$'\t' read -r d dummy h <<< "$riga"
    if [[ "$h" == "version-hidden" ]]; then
        h="$(risolvihashversione "$v")"
    fi
    [[ "$h" != "version-hidden" ]] || {
        errore "L'identificativo storico di questa build non e ancora risolvibile."
        return 1
    }
    esito="$(cdnverifica "$h")"
    cdnsalva "$h" "$esito"
    [[ "$esito" == "disponibile" ]] || {
        errore "Pacchetto non disponibile sul CDN."
        return 1
    }
    zip="$temporanei/prova-$h.zip"
    estrazione="$temporanei/prova-$h"
    cartella="$estrazione/$d $v"
    app="$cartella/RobloxStudio $v.app"
    rm -rf "$estrazione" 2>/dev/null
    mkdir -p "$cartella" 2>/dev/null
    spinneravvia "Calcolo la dimensione della build di prova"
    tot="$(dimensioneremota "$cdnbase/$h-RobloxStudioApp.zip")"
    spinnerferma
    print
    if ! scaricafile "$cdnbase/$h-RobloxStudioApp.zip" "$zip" "$tot"; then
        errore "Download di prova non riuscito."
        rm -f "$zip" 2>/dev/null
        rm -rf "$estrazione" 2>/dev/null
        return 1
    fi
    spinneravvia "Estraggo la build di prova"
    if ! zipvalido "$zip"; then
        spinnerferma
        errore "Il pacchetto di prova non supera i controlli di sicurezza."
        rm -f "$zip" 2>/dev/null
        rm -rf "$estrazione" 2>/dev/null
        return 1
    fi
    if ! ditto -x -k "$zip" "$estrazione/origine" >/dev/null 2>&1; then
        spinnerferma
        errore "Estrazione della build di prova non riuscita."
        rm -f "$zip" 2>/dev/null
        rm -rf "$estrazione" 2>/dev/null
        return 1
    fi
    spinnerferma
    sorgente="$(find "$estrazione/origine" -maxdepth 5 -type d \( -name 'RobloxStudio.app' -o -name 'Roblox Studio.app' \) -print -quit 2>/dev/null)"
    if [[ -z "$sorgente" ]]; then
        errore "Applicazione Roblox Studio non trovata nel pacchetto."
        rm -f "$zip" 2>/dev/null
        rm -rf "$estrazione" 2>/dev/null
        return 1
    fi
    if ! verificaappufficiale "$sorgente" "$v"; then
        errore "Firma Roblox non valida o versione del pacchetto non corrispondente."
        rm -f "$zip" 2>/dev/null
        rm -rf "$estrazione" 2>/dev/null
        return 1
    fi
    mv "$sorgente" "$app" 2>/dev/null
    rm -rf "$estrazione/origine" 2>/dev/null
    rm -f "$zip" 2>/dev/null
    [[ -d "$app" ]] || {
        errore "Preparazione della build di prova non riuscita."
        rm -rf "$estrazione" 2>/dev/null
        return 1
    }
    if ! preparaappmac "$app" "$v"; then
        errore "La verifica finale della firma Roblox non e riuscita."
        rm -rf "$estrazione" 2>/dev/null
        return 1
    fi
    proteggiprova "$app"
    titolo
    testaapp "$app" rapida
    rc=$?
    chiudistudio
    sproteggiprova "$app"
    rm -rf "$estrazione" 2>/dev/null
    return $rc
}

verificabuild() {
    local v="$1" app
    app="$(trovaapp "$v")"
    if [[ -n "$app" && -d "$app" ]]; then
        titolo
        testaapp "$app" rapida
        return $?
    fi
    titolo
    sezione "Verifica temporanea $v"
    nota "La build viene scaricata in Temporanei, testata e poi eliminata."
    verificatemp "$v"
}

listanonoutofdate() {
    local riga d v h raw c f t cdn i
    local -a valide hash selezioni
    valide=()
    for riga in "${(@f)$(cat "$catalogo" 2>/dev/null | sort -t $'\t' -k1,1 -k2,2)}"; do
        [[ -n "$riga" ]] || continue
        IFS=$'\t' read -r d v h <<< "$riga"
        raw="$(statover "$v")"
        [[ "$raw" == "supportata" ]] || continue
        testfresco "$v" "$VERIFICA_TTL" || continue
        c="$(compatver "$v")"
        [[ "$c" == "compatibile" ]] || continue
        if [[ "$h" == "version-hidden" ]]; then
            h="$(risolvihashversione "$v")"
        fi
        [[ -n "$h" && "$h" != "version-hidden" ]] || continue
        valide+=("$d"$'\t'"$v"$'\t'"$h")
    done
    if (( ${#valide} == 0 )); then
        sezione "Scaricabili e funzionali"
        avviso "Nessuna build e ancora confermata valida."
        nota "Usa Trova altre versioni funzionali per costruire l'elenco."
        return 0
    fi
    hash=()
    for riga in "${valide[@]}"; do
        IFS=$'\t' read -r d v h <<< "$riga"
        hash+=("$h")
    done
    nota "Ricontrollo che le build valide siano ancora scaricabili."
    cdnlotto "${hash[@]}"
    SC_RIGHE=()
    selezioni=()
    for (( i=1; i<=${#valide}; i++ )); do
        riga="${valide[i]}"
        IFS=$'\t' read -r d v h <<< "$riga"
        cdn="${CDNRIS[i]}"
        [[ "$cdn" == "disponibile" ]] || continue
        f="$(fasciaversione "$v" "$d")"
        t="$(testover "$v")"
        SC_RIGHE+=("$d"$'\t'"$v"$'\t'"$f"$'\t'"disponibile"$'\t'"compatibile"$'\t'"VALIDA"$'\t'"$t")
        selezioni+=("$riga")
    done
    if (( ${#SC_RIGHE} == 0 )); then
        sezione "Scaricabili e funzionali"
        avviso "Le build confermate valide non risultano scaricabili in questo momento."
        return 0
    fi
    SC_TITOLO="Scaricabili e funzionali"
    SC_SPEC=("N|3|3|0" "Data|10|10|1" "Versione|11|16|0" "Eta|8|10|1" "CDN|3|11|2" "Mac|8|12|3" "Roblox|8|9|0" "Test|10|16|4")
    SC_NOTA="Tutte le build qui presenti sono state realmente provate, risultano scaricabili e non OUT OF DATE."
    scegliriga || return 0
    IFS=$'\t' read -r d v h <<< "${selezioni[SC_SEL]}"
    titolo
    azionebuild "$d" "$v" "$h" "disponibile"
}

listavecchie() {
    local riga d v h s f cdn i
    local -a vecchie hash selezioni cdnsel
    vecchie=()
    for riga in "${(@f)$(cat "$catalogo" 2>/dev/null)}"; do
        [[ -n "$riga" ]] || continue
        IFS=$'\t' read -r d v h <<< "$riga"
        f="$(fasciaversione "$v" "$d")"
        if [[ "$f" == "VECCHIA" || "$f" == "STORICA" ]]; then
            vecchie+=("$riga")
        fi
        (( ${#vecchie} >= 60 )) && break
    done
    if (( ${#vecchie} > 1 )); then
        vecchie=("${(@f)$(printf '%s\n' "${vecchie[@]}" | sort -t $'\t' -k1,1 -k2,2)}")
    fi
    if (( ${#vecchie} == 0 )); then
        avviso "Nessuna versione vecchia nel catalogo."
        return 0
    fi
    hash=()
    for riga in "${vecchie[@]}"; do
        IFS=$'\t' read -r d v h <<< "$riga"
        if [[ "$h" == "version-hidden" ]]; then
            h="$(risolvihashversione "$v")"
        fi
        hash+=("$h")
    done
    nota "Controllo la disponibilita delle versioni vecchie sul CDN ufficiale."
    cdnlotto "${hash[@]}"
    calcolalimiti
    SC_RIGHE=()
    selezioni=()
    cdnsel=()
    for (( i=1; i<=${#vecchie}; i++ )); do
        riga="${vecchie[i]}"
        IFS=$'\t' read -r d v h <<< "$riga"
        cdn="${CDNRIS[i]}"
        [[ -n "$cdn" ]] || cdn="non disponibile"
        s="$(statoroblox "$v" "$d")"
        f="$(fasciaversione "$v" "$d")"
        intelligenzaversione "$v" "$d"
        SC_RIGHE+=("$(datavisibile "$d")"$'\t'"$v"$'\t'"$f"$'\t'"$cdn"$'\t'"$s"$'\t'"$INT_CONF")
        selezioni+=("$riga")
        cdnsel+=("$cdn")
    done
    SC_TITOLO="Versioni vecchie"
    SC_SPEC=("N|3|3|0" "Data|10|10|1" "Versione|11|16|0" "Eta|8|10|0" "CDN|3|15|2" "Roblox|11|13|0" "Conf.|5|6|3")
    SC_NOTA="VECCHIA = 91-180 giorni dalla release di riferimento. STORICA = oltre 180 giorni."
    scegliriga || return 0
    IFS=$'\t' read -r d v h <<< "${selezioni[SC_SEL]}"
    titolo
    azionebuild "$d" "$v" "$h" "${cdnsel[SC_SEL]}"
}

scansioneintelligente() {
    local modalita="${1:-normale}" limite="$FUNZIONALI_SESSIONE"
    local riga d v h raw app esito fatti=0 saltate=0 nondisponibili=0 finale risposta
    local -a tutte
    titolo
    sezione "Ricerca versioni funzionali"
    if [[ "$modalita" == "completa" ]]; then
        limite=0
        avviso "La scansione completa puo scaricare molte build."
        nota "Ogni build viene provata in Temporanei e rimossa dopo il test."
        domanda "Scrivi SCANSIONE per procedere:"
        IFS= read -r risposta || return 0
        [[ "$risposta" == "SCANSIONE" ]] || return 0
    else
        campo "Nuovi test massimi" "$limite"
        nota "La ricerca parte dalle build piu recenti e continua verso le precedenti."
    fi
    spinneravvia "Sincronizzo il catalogo Roblox"
    sincronizzaufficiale >/dev/null 2>&1
    spinnerferma
    tutte=("${(@f)$(cat "$catalogo" 2>/dev/null)}")
    for riga in "${tutte[@]}"; do
        [[ -n "$riga" ]] || continue
        IFS=$'\t' read -r d v h <<< "$riga"
        raw="$(statover "$v")"
        if [[ "$raw" == "supportata" ]] && testfresco "$v" "$VERIFICA_TTL"; then
            (( saltate++ ))
            continue
        fi
        if [[ "$raw" == "obsoleta" ]] && testfresco "$v" "$OOD_TTL"; then
            (( saltate++ ))
            continue
        fi
        if [[ "$raw" == "nonavviabile" ]]; then
            (( saltate++ ))
            continue
        fi
        app="$(trovaapp "$v")"
        if [[ -n "$app" && -d "$app" ]]; then
            preparaappmac "$app"
            titolo
            sezione "Test $(( fatti + 1 ))"
            campo "Data build" "$d"
            campo "Versione" "$v"
            testaapp "$app" rapida
            finale=$?
        else
            if [[ "$h" == "version-hidden" ]]; then
                h="$(risolvihashversione "$v")"
            fi
            if [[ -z "$h" || "$h" == "version-hidden" ]]; then
                (( nondisponibili++ ))
                continue
            fi
            esito="$(cdncache "$h" 2>/dev/null)"
            if [[ -z "$esito" || "$esito" == "rete incerta" || "$esito" == "non verificata" ]]; then
                esito="$(cdnverifica "$h")"
                cdnsalva "$h" "$esito"
            fi
            if [[ "$esito" != "disponibile" ]]; then
                (( nondisponibili++ ))
                continue
            fi
            titolo
            sezione "Test $(( fatti + 1 ))"
            campo "Data build" "$d"
            campo "Versione" "$v"
            verificatemp "$v"
            finale=$?
        fi
        (( fatti++ ))
        if (( limite > 0 && fatti >= limite )); then
            break
        fi
    done
    titolo
    sezione "Ricerca completata"
    campo "Nuovi test" "$fatti"
    campo "Gia classificati" "$saltate"
    campo "Non scaricabili" "$nondisponibili"
    nota "VALIDA e OUT OF DATE vengono assegnati solo dopo una prova reale."
}

azionebuild() {
    local d="$1" v="$2" h="$3" cdn="$4" s scelta
    sezione "Versione $v"
    calcolalimiti
    s="$(statoroblox "$v" "$d")"
    intelligenzaversione "$v" "$d"
    campo "Data build" "$(datavisibile "$d")"
    campo "Eta build" "$(fasciaversione "$v" "$d")"
    campo "Disponibilita CDN" "$cdn"
    campo "Stato Roblox" "$s"
    campo "Valutazione" "$INT_ESITO"
    campo "Affidabilita" "$INT_CONF"
    campo "Fonte stato" "$INT_FONTE"
    print
    voce 1 "Scarica o riscarica"
    voce 2 "Verifica out of date"
    voce 0 "Torna indietro"
    print
    domanda "Scelta:"
    IFS= read -r scelta || return 0
    case "$scelta" in
        (1)
            if [[ "$s" == "OUT OF DATE" ]]; then
                avviso "Roblox risulta rifiutare questa build."
                domanda "Scaricarla comunque [s/N]:"
                IFS= read -r scelta || return 0
                [[ "$scelta" == [sSyY] ]] || return 0
            fi
            scaricaversione "$v"
            ;;
        (2) verificabuild "$v" ;;
        (*) ;;
    esac
}

contafunzionali() {
    local d v statoletto resto testdata n=0
    while IFS=$'\t' read -r d v statoletto resto testdata; do
        [[ "$statoletto" == "supportata" ]] || continue
        testfresco "$v" "$VERIFICA_TTL" || continue
        [[ "$(compatver "$v")" == "compatibile" ]] || continue
        (( n++ ))
    done < "$stato"
    print -r -- "$n"
}

panoramaversioni() {
    local corrente totale funzionali ood registrati installate riga d v s c
    local piurecente="" piurecentedata="" piuvecchia="" piuvecchiadata=""
    titolo
    sezione "Panoramica versioni"
    catalogopronto || return 0
    spinneravvia "Aggiorno lo stato generale"
    if reteattiva; then
        corrente="$(versionecorrente)"
    else
        corrente="$(awk -F '\t' 'NR==1 {print $1}' "$correnteweb" 2>/dev/null)"
    fi
    totale="$(awk -F '\t' 'NF>=2 {n++} END {print n+0}' "$catalogo" 2>/dev/null)"
    funzionali="$(contafunzionali)"
    ood=0
    registrati=0
    while IFS=$'\t' read -r d v s _ _; do
        [[ -n "$v" ]] || continue
        (( registrati++ ))
        if [[ "$s" == "obsoleta" ]] && testfresco "$v" "$OOD_TTL"; then
            (( ood++ ))
        fi
        if [[ "$s" == "supportata" ]] && testfresco "$v" "$VERIFICA_TTL" && [[ "$(compatver "$v")" == "compatibile" ]]; then
            if [[ -z "$piuvecchiadata" || "$d" < "$piuvecchiadata" ]]; then
                piuvecchiadata="$d"
                piuvecchia="$v"
            fi
            if [[ -z "$piurecentedata" || "$d" > "$piurecentedata" ]]; then
                piurecentedata="$d"
                piurecente="$v"
            fi
        fi
    done < "$stato"
    elencoapp
    installate="${#APPS}"
    spinnerferma
    [[ "$corrente" == 0.* ]] && campo "Versione live" "$corrente"
    campo "Build catalogate" "$totale"
    campo "Funzionali verificate" "$funzionali"
    campo "OUT OF DATE recenti" "$ood"
    campo "Test registrati" "$registrati"
    campo "Versioni installate" "$installate"
    if [[ -n "$piurecente" ]]; then
        campo "Funzionale piu recente" "$piurecente"
        campo "Data piu recente" "$(datavisibile "$piurecentedata")"
    fi
    if [[ -n "$piuvecchia" ]]; then
        campo "Funzionale piu vecchia" "$piuvecchia"
        campo "Data piu vecchia" "$(datavisibile "$piuvecchiadata")"
    fi
    print
    nota "Una build entra tra le funzionali solo dopo un test reale recente e una compatibilita Mac confermata."
}

confrontaversioni() {
    local a b d h rawh cdn s c t fonte scelta
    local -a versioni dati
    catalogopronto || return 0
    titolo
    sezione "Confronta versioni"
    domanda "Prima versione:"
    IFS= read -r a || return 0
    a="$(print -r -- "$a" | tr -d '[:space:]')"
    versionevalida "$a" || { errore "Prima versione non valida."; return 0; }
    domanda "Seconda versione:"
    IFS= read -r b || return 0
    b="$(print -r -- "$b" | tr -d '[:space:]')"
    versionevalida "$b" || { errore "Seconda versione non valida."; return 0; }
    [[ "$a" != "$b" ]] || { avviso "Scegli due versioni diverse."; return 0; }
    versioni=("$a" "$b")
    dati=()
    spinneravvia "Raccolgo disponibilita e stato"
    for v in "${versioni[@]}"; do
        d="$(dataversione "$v")"
        [[ -n "$d" ]] || { spinnerferma; errore "La versione $v non e presente nel catalogo."; return 0; }
        rawh="$(hashversione "$v" 2>/dev/null)"
        fonte="catalogo Roblox"
        h="$rawh"
        if [[ "$h" == "version-hidden" ]]; then
            h="$(risolvihashversione "$v")"
            if hashvalido "$h"; then
                fonte="resolver storico"
            else
                fonte="hash non disponibile"
            fi
        fi
        if hashvalido "$h"; then
            cdn="$(cdncache "$h" 2>/dev/null)"
            if [[ -z "$cdn" ]]; then
                cdn="$(cdnverifica "$h")"
                cdnsalva "$h" "$cdn" >/dev/null 2>&1 || true
            fi
        else
            cdn="non verificata"
        fi
        s="$(statoroblox "$v" "$d")"
        c="$(compatver "$v")"
        [[ "$c" == "sconosciuta" || -z "$c" ]] && c="non verificata"
        t="$(testover "$v")"
        dati+=("$d"$'\t'"$v"$'\t'"$(fasciaversione "$v" "$d")"$'\t'"$cdn"$'\t'"$c"$'\t'"$s"$'\t'"$t"$'\t'"$fonte")
    done
    spinnerferma
    titolo
    sezione "Confronto"
    for riga in "${dati[@]}"; do
        IFS=$'\t' read -r d v eta cdn c s t fonte <<< "$riga"
        campo "Versione" "$v"
        campo "Data build" "$(datavisibile "$d")"
        campo "Eta" "$eta"
        campo "CDN" "$cdn"
        campo "Compatibilita" "$c"
        campo "Stato Roblox" "$s"
        campo "Ultimo test" "$t"
        campo "Origine hash" "$fonte"
        print
    done
    nota "La disponibilita del pacchetto non equivale alla compatibilita o all'accettazione della build da parte di Roblox."
}

menuricercaversioni() {
    local scelta
    while true; do
        titolo
        sezione "Ricerca versioni"
        voce 1 "Ricerca automatica"
        voce 2 "Scansione completa"
        voce 3 "Sincronizza catalogo"
        voce 0 "Torna indietro"
        print
        nota "La ricerca automatica prova un numero limitato di nuove build; la scansione completa procede senza limite."
        print
        domanda "Scelta:"
        IFS= read -r scelta || return 0
        case "$scelta" in
            (1) scansioneintelligente normale; pausa ;;
            (2) scansioneintelligente completa; pausa ;;
            (3) titolo; aggiornacatalogo; pausa ;;
            (0) return 0 ;;
            (*) ;;
        esac
    done
}

menuversioni() {
    local scelta n
    n="$(contafunzionali)"
    if [[ "$n" == "0" ]]; then
        titolo
        sezione "Versioni"
        info "Non esistono ancora build confermate. Avvio una prima ricerca automatica."
        scansioneintelligente normale
        pausa
    fi
    while true; do
        titolo
        sezione "Versioni"
        voce 1 "Panoramica"
        voce 2 "Scaricabili e funzionali"
        voce 3 "Ricerca versioni"
        voce 4 "Verifica una versione"
        voce 5 "Confronta versioni"
        voce 0 "Torna indietro"
        print
        nota "Scoperta, verifica e confronto delle build sono raccolti in questa sezione."
        print
        domanda "Scelta:"
        IFS= read -r scelta || return 0
        case "$scelta" in
            (1) panoramaversioni; pausa ;;
            (2) titolo; listanonoutofdate; pausa ;;
            (3) menuricercaversioni ;;
            (4) titolo; cercacatalogo; pausa ;;
            (5) confrontaversioni; pausa ;;
            (0) return 0 ;;
            (*) ;;
        esac
    done
}

permessodownload() {
    local prova="$HOME/Downloads/.robloxstudiopermesso.$$"
    if : > "$prova" 2>/dev/null; then
        rm -f "$prova" 2>/dev/null
        return 0
    fi
    rm -f "$prova" 2>/dev/null
    return 1
}

permessoautomazione() {
    conlimite 6 osascript >/dev/null 2>&1 <<'FINE'
tell application "System Events"
    get name of first process whose frontmost is true
end tell
FINE
}

apripermesso() {
    case "$1" in
        (file) open "x-apple.systempreferences:com.apple.preference.security?Privacy_FilesAndFolders" >/dev/null 2>&1 || true ;;
        (automazione) open "x-apple.systempreferences:com.apple.preference.security?Privacy_Automation" >/dev/null 2>&1 || true ;;
        (accessibilita) open "x-apple.systempreferences:com.apple.preference.security?Privacy_Accessibility" >/dev/null 2>&1 || true ;;
    esac
}

richiedipermessi() {
    local fd au ac risposta
    while true; do
        titolo
        sezione "Permessi obbligatori"
        if permessodownload; then fd="abilitato"; else fd="mancante"; fi
        if permessoautomazione; then au="abilitata"; else au="mancante"; fi
        if [[ "$(accessibilita)" == "si" ]]; then ac="abilitata"; else ac="mancante"; fi
        campo "Download" "$fd"
        campo "Automazione" "$au"
        campo "Accessibilita" "$ac"
        if [[ "$fd" == "abilitato" && "$au" == "abilitata" && "$ac" == "abilitata" ]]; then
            print
            ok "Permessi necessari disponibili."
            sleep 1
            return 0
        fi
        print
        avviso "Senza questi permessi la verifica delle versioni non e affidabile."
        if [[ "$fd" != "abilitato" ]]; then
            apripermesso file
            info "Concedi a Terminale l'accesso alla cartella Download."
        elif [[ "$au" != "abilitata" ]]; then
            apripermesso automazione
            info "Consenti a Terminale di controllare System Events."
        elif [[ "$ac" != "abilitata" ]]; then
            apripermesso accessibilita
            info "Abilita Terminale in Accessibilita."
        fi
        print
        domanda "Concedi il permesso, poi premi Invio. 0 per uscire:"
        IFS= read -r risposta || exit 1
        [[ "$risposta" == "0" ]] && exit 1
    done
}

menuarchivio() {
    local scelta v
    while true; do
        titolo
        sezione "Archivio"
        voce 1 "Versioni storiche scaricabili"
        voce 2 "Cerca una build storica"
        voce 3 "Importa una copia locale"
        voce 0 "Torna indietro"
        print
        nota "L'Archivio e separato dalle build confermate funzionali. Una copia storica puo essere OUT OF DATE."
        print
        domanda "Scelta:"
        IFS= read -r scelta || return 0
        case "$scelta" in
            (1) titolo; listavecchie; pausa ;;
            (2) titolo; cercacatalogo; pausa ;;
            (3)
                titolo
                sezione "Importa copia locale"
                domanda "Versione:"
                IFS= read -r v || continue
                v="$(print -r -- "$v" | tr -d '[:space:]')"
                installadaarchivio "$v"
                pausa
                ;;
            (0) return 0 ;;
            (*) ;;
        esac
    done
}

accessibilita() {
    conlimite 6 osascript 2>/dev/null <<'FINE'
tell application "System Events"
    if UI elements enabled then
        return "si"
    else
        return "no"
    end if
end tell
FINE
}

testoui() {
    conlimite 8 osascript 2>/dev/null <<'FINE'
tell application "System Events"
    if not UI elements enabled then return "__NOACCESS__"
    set p to missing value
    set candidati to {"RobloxStudio", "RobloxStudioBeta", "Roblox Studio"}
    repeat with nomeprocesso in candidati
        try
            if exists application process (nomeprocesso as text) then
                set p to application process (nomeprocesso as text)
                exit repeat
            end if
        end try
    end repeat
    if p is missing value then return "__NOPROC__"
    set risultato to ""
    try
        repeat with w in windows of p
            try
                set risultato to risultato & (name of w as text) & linefeed
            end try
            try
                repeat with e in entire contents of w
                    try
                        set n to name of e
                        if n is not missing value then set risultato to risultato & (n as text) & linefeed
                    end try
                    try
                        set v to value of e
                        if v is not missing value then set risultato to risultato & (v as text) & linefeed
                    end try
                    try
                        set d to description of e
                        if d is not missing value then set risultato to risultato & (d as text) & linefeed
                    end try
                end repeat
            end try
        end repeat
    end try
    return risultato
end tell
FINE
}

chiudipannello() {
    conlimite 6 osascript >/dev/null 2>&1 <<'FINE'
tell application "System Events"
    if not UI elements enabled then return
    set p to missing value
    set candidati to {"RobloxStudio", "RobloxStudioBeta", "Roblox Studio"}
    repeat with nomeprocesso in candidati
        try
            if exists application process (nomeprocesso as text) then
                set p to application process (nomeprocesso as text)
                exit repeat
            end if
        end try
    end repeat
    if p is missing value then return
    try
        set frontmost of p to true
        key code 53
    end try
end tell
FINE
}

processoattivo() {
    pgrep -f "/RobloxStudio[^/]*/Contents/MacOS/" >/dev/null 2>&1 || pgrep -x RobloxStudio >/dev/null 2>&1 || pgrep -x RobloxStudioBeta >/dev/null 2>&1
}

chiudistudio() {
    killall RobloxStudio 2>/dev/null
    killall RobloxStudioBeta 2>/dev/null
}

cercalog() {
    local marker="$1" output="$2" f
    : > "$output"
    [[ -d "$logroot" ]] || return 0
    for f in "$logroot"/**/*(N.); do
        [[ "$f" -nt "$marker" ]] && tail -n 500 "$f" >> "$output" 2>/dev/null
    done
}

testaapp() {
    local app="$1" modalita="${2:-completa}"
    local durataprova exe marker logtest uitest syslog pattern uivista=0 processo=0 inizio now trascorso=0
    if [[ "$modalita" == "rapida" ]]; then
        durataprova=12
    else
        durataprova=26
    fi
    datiapp "$app"
    sezione "Verifica out of date"
    if ! verificaappufficiale "$app" "$VERS"; then
        salvastato "$DATA" "$VERS" "nonavviabile" "firma Roblox non valida o applicazione modificata"
        errore "Verifica bloccata: firma Roblox non valida o applicazione modificata."
        return 2
    fi
    campo "Data build" "$(datavisibile "$DATA")"
    campo "Versione" "$VERS"
    compatapp "$app" "no" >/dev/null 2>&1
    local c="$(compatver "$VERS")"
    if [[ "$c" == "macos vecchio" || "$c" == "cpu" || "$c" == "non avviabile" ]]; then
        salvastato "$DATA" "$VERS" "nonavviabile" "incompatibile con questo Mac: $c"
        print
        errore "Non avviabile su questo Mac: $c"
        return 2
    fi
    exe="$(find "$app/Contents/MacOS" -maxdepth 1 -type f -print -quit 2>/dev/null)"
    if [[ -z "$exe" ]]; then
        salvastato "$DATA" "$VERS" "nonavviabile" "eseguibile non trovato"
        print
        errore "Il pacchetto e incompleto: eseguibile non trovato."
        return 2
    fi
    if [[ "$(accessibilita)" != "si" ]]; then
        salvastato "$DATA" "$VERS" "incerta" "accessibilita del Terminale non abilitata"
        print
        avviso "Il Terminale non puo leggere l'interfaccia di Roblox Studio."
        info "Apri Impostazioni di Sistema, poi Privacy e Sicurezza, poi Accessibilita e abilita Terminale."
        nota "Senza questo permesso la versione non puo essere dichiarata valida."
        return 4
    fi
    chiudistudio
    sleep 1
    marker="$temporanei/marker"
    logtest="$temporanei/logtest.txt"
    uitest="$temporanei/uitest.txt"
    syslog="$temporanei/syslog.txt"
    rm -f "$marker" "$logtest" "$uitest" "$syslog" 2>/dev/null
    touch "$marker"
    : > "$syslog"
    /usr/bin/log stream --style compact --level info --predicate 'process == "RobloxStudio" OR process == "RobloxStudioBeta"' > "$syslog" 2>/dev/null &!
    LOGPID=$!
    print
    if ! open -na "$app" >/dev/null 2>&1; then
        kill "$LOGPID" 2>/dev/null
        LOGPID=""
        salvastato "$DATA" "$VERS" "nonavviabile" "apertura rifiutata dal sistema"
        errore "macOS ha rifiutato l'apertura di questa versione."
        return 3
    fi
    pattern='out[ -]?of[ -]?date|outdated|version[^[:cntrl:]]*(too old|no longer supported)|must[^[:cntrl:]]*update|need[^[:cntrl:]]*to[^[:cntrl:]]*update|update[^[:cntrl:]]*(required|needed|mandatory)|please[^[:cntrl:]]*update|unsupported[^[:cntrl:]]*version|client[^[:cntrl:]]*too old|versione[^[:cntrl:]]*(obsoleta|non supportata)|aggiornamento[^[:cntrl:]]*(obbligatorio|richiesto)'
    [[ -t 1 ]] && printf '\033[?25l'
    inizio="$(date +%s)"
    while true; do
        now="$(date +%s)"
        trascorso=$(( now - inizio ))
        (( trascorso >= durataprova )) && break
        [[ -t 1 ]] && barra $(( trascorso * 100 / durataprova )) "analisi avvio $trascorso/$durataprova s"
        testoui > "$uitest" 2>/dev/null
        if [[ -s "$uitest" ]] && ! grep -Eq '^__(NOACCESS|NOPROC)__$' "$uitest"; then
            uivista=1
        fi
        cercalog "$marker" "$logtest"
        if grep -Eia "$pattern" "$uitest" "$logtest" "$syslog" >/dev/null 2>&1; then
            kill "$LOGPID" 2>/dev/null
            LOGPID=""
            spinnerferma
            salvastato "$DATA" "$VERS" "obsoleta" "avviso di aggiornamento obbligatorio rilevato"
            print
            errore "OUT OF DATE: Roblox impone l'aggiornamento, questa versione non e utilizzabile."
            chiudipannello
            chiudistudio
            return 1
        fi
        if processoattivo; then
            processo=1
        elif (( trascorso >= 8 )); then
            break
        fi
        sleep 1
    done
    kill "$LOGPID" 2>/dev/null
    LOGPID=""
    spinnerferma
    cercalog "$marker" "$logtest"
    if grep -Eia "$pattern" "$uitest" "$logtest" "$syslog" >/dev/null 2>&1; then
        salvastato "$DATA" "$VERS" "obsoleta" "avviso rilevato al controllo finale"
        print
        errore "OUT OF DATE: rilevato avviso di aggiornamento al controllo finale."
        chiudistudio
        return 1
    fi
    if (( uivista == 1 )) && (( processo == 1 )) && processoattivo; then
        salvastato "$DATA" "$VERS" "supportata" "interfaccia leggibile e processo stabile"
        print
        ok "VALIDA: interfaccia caricata, nessun avviso di aggiornamento, processo stabile."
        chiudistudio
        return 0
    fi
    salvastato "$DATA" "$VERS" "incerta" "segnali insufficienti durante la prova"
    print
    avviso "INCERTA: la prova non ha raccolto abbastanza segnali per dichiararla valida."
    nota "Riprova tenendo la finestra di Roblox Studio in primo piano."
    chiudistudio
    return 4
}

menuverificaselezione() {
    selezionaapp "Scegli la versione da verificare" || return 0
    titolo
    testaapp "$APPSCELTA"
}

menuverificatutte() {
    local app
    local -a lista
    righeinstallate
    if (( ${#APPS} == 0 )); then
        avviso "Nessuna versione installata."
        return 0
    fi
    lista=("${APPS[@]}")
    for app in "${lista[@]}"; do
        titolo
        testaapp "$app"
        sleep 1
    done
    print
    ok "Verifica completata su ${#lista} versioni."
}

menuverificamancanti() {
    local app s
    local -a lista
    righeinstallate
    lista=()
    for app in "${APPS[@]}"; do
        datiapp "$app"
        s="$(statover "$VERS")"
        [[ "$s" == "noncontrollata" || "$s" == "incerta" ]] && lista+=("$app")
    done
    if (( ${#lista} == 0 )); then
        ok "Tutte le versioni installate sono gia state verificate."
        return 0
    fi
    for app in "${lista[@]}"; do
        titolo
        testaapp "$app"
        sleep 1
    done
    print
    ok "Verifica completata su ${#lista} versioni."
}

menupiuvecchia() {
    local app migliore="" miglioredata="" s c
    righeinstallate
    sezione "Versione utilizzabile piu vecchia"
    for app in "${APPS[@]}"; do
        datiapp "$app"
        c="$(compatver "$VERS")"
        [[ "$c" == "compatibile" ]] || continue
        s="$(statoroblox "$VERS" "$DATA")"
        case "$s" in
            (corrente|valida|"valida*")
                if [[ -z "$miglioredata" || "$DATA" < "$miglioredata" ]]; then
                    migliore="$app"
                    miglioredata="$DATA"
                fi
                ;;
        esac
    done
    if [[ -z "$migliore" ]]; then
        avviso "Nessuna versione risulta insieme compatibile con il Mac e accettata da Roblox."
        nota "Esegui le verifiche di avvio per popolare i dati."
        return 0
    fi
    datiapp "$migliore"
    campo "Data build" "$(datavisibile "$DATA")"
    campo "Versione" "$VERS"
    campo "Stato Roblox" "$(statoroblox "$VERS" "$DATA")"
    campo "Compatibilita" "$(compatver "$VERS")"
}

menucompatibilita() {
    local app scelta
    sezione "Compatibilita con questo Mac"
    campo "macOS" "$SISTEMA"
    campo "Processore" "$ARCHMAC"
    print
    voce 1 "Ricontrolla tutte le versioni installate"
    voce 2 "Dettaglio di una versione"
    voce 0 "Torna indietro"
    print
    domanda "Scelta:"
    IFS= read -r scelta || return 0
    case "$scelta" in
        (1)
            righeinstallate
            if (( ${#APPS} == 0 )); then
                avviso "Nessuna versione installata."
                return 0
            fi
            local i=0
            [[ -t 1 ]] && printf '\033[?25l'
            for app in "${APPS[@]}"; do
                (( i++ ))
                datiapp "$app"
                [[ -t 1 ]] && barra $(( i * 100 / ${#APPS} )) "analisi $VERS"
                compatapp "$app" "no" >/dev/null 2>&1
            done
            spinnerferma
            titolo
            mostrainstallate
            ;;
        (2)
            selezionaapp "Scegli la versione da analizzare" || return 0
            titolo
            compatapp "$APPSCELTA" "si"
            ;;
        (*) ;;
    esac
}

menuverificheinstallate() {
    local scelta
    while true; do
        titolo
        sezione "Verifica installate"
        voce 1 "Verifica una versione"
        voce 2 "Verifica quelle non controllate"
        voce 3 "Verifica tutte"
        voce 0 "Torna indietro"
        print
        nota "La verifica apre Roblox Studio, legge interfaccia e log e poi lo chiude."
        print
        domanda "Scelta:"
        IFS= read -r scelta || return 0
        case "$scelta" in
            (1) menuverificaselezione; pausa ;;
            (2) titolo; menuverificamancanti; pausa ;;
            (3) menuverificatutte; pausa ;;
            (0) return 0 ;;
            (*) ;;
        esac
    done
}

apriapp() {
    local app="$1" conferma
    datiapp "$app"
    if ! preparaappmac "$app" "$VERS"; then
        errore "Avvio bloccato: firma Roblox non valida o applicazione modificata."
        return 1
    fi
    compatassicura "$app" "$VERS"
    calcolalimiti
    local codice
    verdetto "$VERS" "$DATA"
    codice=$?
    sezione "Avvio versione $VERS"
    campo "Data build" "$(datavisibile "$DATA")"
    campo "Esito controllo" "$VERDETTO"
    info "$MOTIVO"
    print
    if (( codice == 1 )); then
        errore "Avvio annullato: questa build non puo funzionare su questo Mac."
        return 1
    fi
    if (( codice == 2 )); then
        avviso "Roblox rifiutera questa build e mostrera la richiesta di aggiornamento."
        nota "Modalita archivio: la build puo essere aperta e protetta dagli aggiornamenti, ma il rifiuto server OUT OF DATE non viene aggirato."
        print
        domanda "Aprirla lo stesso [s/N]:"
        IFS= read -r conferma || return 1
        [[ "$conferma" == [sSyY] ]] || return 1
    fi
    if (( codice == 3 )); then
        nota "Nessuna verifica registrata: l'apertura procede ma l'esito non e garantito."
    fi
    spinneravvia "Blocco gli aggiornamenti automatici"
    bloccaapp "$app"
    spinnerferma
    spinneravvia "Avvio Roblox Studio"
    if ! open -na "$app" >/dev/null 2>&1; then
        spinnerferma
        errore "macOS ha rifiutato l'apertura dell'applicazione."
        return 1
    fi
    sleep 2
    spinnerferma
    ok "Roblox Studio $VERS avviato."
    return 0
}

avviorapido() {
    local migliore corrente
    titolo
    sezione "Avvio rapido"
    spinneravvia "Controllo la connessione ai server Roblox"
    if reteattiva; then
        spinnerferma
        spinneravvia "Leggo la versione pubblicata da Roblox"
        corrente="$(versionecorrente)"
        spinnerferma
        [[ -n "$corrente" ]] && campo "Versione live" "$corrente"
    else
        spinnerferma
        avviso "Nessuna connessione: uso soltanto i dati gia salvati."
    fi
    spinneravvia "Valuto le versioni installate"
    migliore="$(migliorescelta)"
    spinnerferma
    if [[ -z "$migliore" ]]; then
        avviso "Nessuna versione installata risulta insieme compatibile e accettata da Roblox."
        info "Usa Installate per verificare le versioni presenti, oppure Versioni per cercarne una nuova."
        return 1
    fi
    apriapp "$migliore"
}

menuinstallate() {
    local scelta
    while true; do
        titolo
        if ! mostrainstallate; then
            print
            voce 1 "Vai a Versioni"
            voce 0 "Torna indietro"
            print
            domanda "Scelta:"
            IFS= read -r scelta || return 0
            [[ "$scelta" == "1" ]] && menuversioni
            [[ "$scelta" == "0" ]] && return 0
            continue
        fi
        voce 1 "Avvio rapido"
        voce 2 "Apri una versione"
        voce 3 "Verifica installate"
        voce 4 "Compatibilita con questo Mac"
        voce 5 "Blocco aggiornamenti"
        voce 6 "Rimuovi una versione"
        voce 0 "Torna indietro"
        print
        domanda "Scelta:"
        IFS= read -r scelta || return 0
        case "$scelta" in
            (1) avviorapido; pausa ;;
            (2)
                selezionaapp "Scegli la versione da aprire" || continue
                titolo
                apriapp "$APPSCELTA"
                pausa
                ;;
            (3) menuverificheinstallate ;;
            (4) titolo; menucompatibilita; pausa ;;
            (5) menublocco; pausa ;;
            (6)
                selezionaapp "Scegli la versione da rimuovere" || continue
                titolo
                rimuoviuna "$APPSCELTA"
                pausa
                ;;
            (0) return 0 ;;
            (*) ;;
        esac
    done
}

menublocco() {
    local scelta app
    titolo
    sezione "Blocco aggiornamenti"
    info "Il blocco impedisce a Roblox Studio di sovrascriversi con la versione piu recente."
    print
    voce 1 "Blocca tutte le versioni installate"
    voce 2 "Sblocca tutte le versioni installate"
    voce 3 "Blocca una versione"
    voce 4 "Sblocca una versione"
    voce 0 "Torna indietro"
    print
    domanda "Scelta:"
    IFS= read -r scelta || return 0
    case "$scelta" in
        (1)
            elencoapp
            spinneravvia "Blocco in corso"
            for app in "${APPS[@]}"; do
                bloccaapp "$app"
            done
            spinnerferma
            ok "Bloccate ${#APPS} versioni."
            ;;
        (2)
            elencoapp
            spinneravvia "Sblocco in corso"
            for app in "${APPS[@]}"; do
                sbloccaapp "$app"
            done
            spinnerferma
            ok "Sbloccate ${#APPS} versioni."
            ;;
        (3)
            selezionaapp "Scegli la versione da bloccare" || return 0
            bloccaapp "$APPSCELTA"
            ok "Versione bloccata."
            ;;
        (4)
            selezionaapp "Scegli la versione da sbloccare" || return 0
            sbloccaapp "$APPSCELTA"
            ok "Versione sbloccata."
            ;;
        (*) ;;
    esac
}

rimuovirecord() {
    local v="$1" file tmp
    for file in "$stato" "$compatibilita" "$blocco"; do
        [[ -f "$file" ]] || continue
        tmp="$temporanei/record.tmp"
        awk -F '\t' -v v="$v" '$2!=v {print}' "$file" > "$tmp" 2>/dev/null
        mv "$tmp" "$file" 2>/dev/null
    done
}

rimuoviuna() {
    local app="$1" cartella conferma
    datiapp "$app"
    sezione "Rimozione versione $VERS"
    campo "Data build" "$(datavisibile "$DATA")"
    campo "Cartella" "${app:h:t}"
    print
    avviso "L'operazione elimina definitivamente questa versione dal disco."
    print
    domanda "Scrivi ELIMINA per confermare:"
    IFS= read -r conferma || return 1
    if [[ "$conferma" != "ELIMINA" ]]; then
        nota "Operazione annullata."
        return 1
    fi
    cartella="${app:h}"
    spinneravvia "Rimozione in corso"
    sbloccaapp "$app"
    rm -rf "$cartella" 2>/dev/null
    spinnerferma
    if [[ -e "$cartella" ]]; then
        errore "Rimozione non riuscita: controlla i permessi della cartella."
        return 1
    fi
    rimuovirecord "$VERS"
    ok "Versione $VERS rimossa."
    return 0
}

rimuovigruppo() {
    local tipo="$1" conferma v app i elemento motivo
    local -a righe
    righe=()
    calcolalimiti
    elencoapp
    for app in "${APPS[@]}"; do
        datiapp "$app"
        if [[ "$tipo" == "obsolete" ]]; then
            case "$(statoroblox "$VERS" "$DATA")" in
                ("OUT OF DATE"|"OUT OF DATE*") righe+=("$DATA"$'\t'"$VERS"$'\t'"OUT OF DATE") ;;
            esac
        else
            case "$(compatver "$VERS")" in
                ("macos vecchio"|cpu|"non avviabile") righe+=("$DATA"$'\t'"$VERS"$'\t'"$(compatver "$VERS")") ;;
            esac
        fi
    done
    if (( ${#righe} == 0 )); then
        ok "Nessuna versione da rimuovere."
        return 0
    fi
    if [[ "$tipo" == "obsolete" ]]; then
        sezione "Versioni rifiutate da Roblox"
    else
        sezione "Versioni incompatibili con il Mac"
    fi
    SC_SPEC=("N|3|3|0" "Data|10|10|1" "Versione|11|16|0" "Motivo|11|16|0")
    tabinit "${SC_SPEC[@]}"
    tabbordo
    tabtesta
    tabbordo
    local -a campi
    for (( i=1; i<=${#righe}; i++ )); do
        elemento="${righe[i]}"
        campi=("${(@ps:\t:)elemento}")
        tabriga "$i" "${campi[@]}"
    done
    tabbordo
    print
    avviso "L'operazione elimina definitivamente ${#righe} versioni dal disco."
    print
    domanda "Scrivi ELIMINA per confermare:"
    IFS= read -r conferma || return 1
    if [[ "$conferma" != "ELIMINA" ]]; then
        nota "Operazione annullata."
        return 1
    fi
    [[ -t 1 ]] && printf '\033[?25l'
    for (( i=1; i<=${#righe}; i++ )); do
        IFS=$'\t' read -r elemento v motivo <<< "${righe[i]}"
        app="$(trovaapp "$v")"
        barra $(( i * 100 / ${#righe} )) "rimozione $v"
        if [[ -n "$app" ]]; then
            sbloccaapp "$app"
            rm -rf "${app:h}" 2>/dev/null
            rimuovirecord "$v"
        fi
    done
    spinnerferma
    ok "Rimozione completata."
}

puliscitemporanei() {
    local f
    mkdir -p "$temporanei" 2>/dev/null
    for f in "$temporanei"/*(N) "$temporanei"/.*(N); do
        case "${f:t}" in
            (.|..) ;;
            (*) rm -rf "$f" 2>/dev/null ;;
        esac
    done
}

puliscidownload() {
    local f trovati=0 nome
    local -a nomi
    nomi=(aggiornaordine.command ripararobloxstudio.command unificarobloxstudio.command aggiornacompatibilita.command aggiornainterfaccia.command aggiornapulizia.command aggiornastrumenti.command aggiornaverifica.command)
    for nome in "${nomi[@]}"; do
        f="$HOME/Downloads/$nome"
        if [[ -f "$f" && ! -L "$f" ]]; then
            rm -f "$f" 2>/dev/null
            trovati=1
        fi
    done
    if (( trovati == 0 )); then
        ok "Nessun comando temporaneo del manager da rimuovere."
    else
        ok "Comandi temporanei del manager rimossi."
    fi
}

puliscilog() {
    local f
    if [[ -L "$logroot" ]]; then
        errore "Pulizia log annullata: il percorso dei log e un collegamento simbolico."
        return 1
    fi
    if [[ -d "$logroot" ]]; then
        for f in "$logroot"/*(N) "$logroot"/.*(N); do
            case "${f:t}" in
                (.|..) ;;
                (*) rm -rf "$f" 2>/dev/null ;;
            esac
        done
    fi
    mkdir -p "$logroot" 2>/dev/null
    ok "Log di Roblox Studio azzerati."
}

ordina() {
    local cartella nome d resto nuova nuovo file tmp
    for cartella in "$versioni"/*(N/); do
        nome="${cartella:t}"
        d="${nome%% *}"
        resto="${nome#* }"
        nuova="$(normalizzadata "$d")"
        if [[ -n "$nuova" && "$nuova" != "$d" ]]; then
            nuovo="$versioni/$nuova $resto"
            [[ -e "$nuovo" ]] || mv "$cartella" "$nuovo" 2>/dev/null
        fi
    done
    for file in "$catalogo" "$stato" "$blocco" "$compatibilita"; do
        [[ -f "$file" ]] || continue
        tmp="$temporanei/ordine.tmp"
        awk -F '\t' 'BEGIN{OFS="\t"}
        {
            split($1,a,"-")
            if(length(a[1])==4 && a[2] ~ /^[0-9]+$/ && a[3] ~ /^[0-9]+$/){
                $1=sprintf("%04d-%02d-%02d",a[1],a[2],a[3])
            }
            print
        }' "$file" | sort -t $'\t' -k1,1r -k2,2r > "$tmp" 2>/dev/null
        mv "$tmp" "$file" 2>/dev/null
    done
    preparadati
    ok "Cartelle e archivi riordinati."
}

menupulizia() {
    local scelta
    while true; do
        titolo
        sezione "Pulizia"
        voce 1 "File temporanei"
        voce 2 "Comandi temporanei del manager in Download"
        voce 3 "Log di Roblox Studio"
        voce 4 "Versioni rifiutate da Roblox"
        voce 5 "Versioni incompatibili con il Mac"
        voce 6 "Pulizia completa"
        voce 0 "Torna indietro"
        print
        domanda "Scelta:"
        IFS= read -r scelta || return 0
        case "$scelta" in
            (1) print; spinneravvia "Pulizia temporanei"; puliscistrumenti; puliscitemporanei; spinnerferma; ok "Temporanei rimossi."; pausa ;;
            (2) print; puliscidownload; pausa ;;
            (3) print; puliscilog; pausa ;;
            (4) titolo; rimuovigruppo obsolete; pausa ;;
            (5) titolo; rimuovigruppo incompatibili; pausa ;;
            (6)
                print
                spinneravvia "Pulizia completa in corso"
                puliscistrumenti
                puliscitemporanei
                spinnerferma
                puliscidownload
                puliscilog
                ok "Sistema ripulito."
                pausa
                ;;
            (0) return 0 ;;
            (*) ;;
        esac
    done
}

controllaaggiornamento() {
    local tmp="$temporanei/github-release.json" ultima risposta
    sezione "Aggiornamento manager"
    spinneravvia "Controllo la release stabile su GitHub"
    if ! curlhttps -fsSL --connect-timeout 5 --max-time 15 --retry 1 'https://api.github.com/repos/scarcellagb/robloxstudiomanager/releases/latest' -o "$tmp" 2>/dev/null; then
        spinnerferma
        errore "Non riesco a controllare GitHub in questo momento."
        rm -f "$tmp" 2>/dev/null
        return 1
    fi
    spinnerferma
    ultima="$(sed -nE 's/^[[:space:]]*"tag_name":[[:space:]]*"v?([0-9]+\.[0-9]+\.[0-9]+)".*/\1/p' "$tmp" | head -n 1)"
    rm -f "$tmp" 2>/dev/null
    if [[ ! "$ultima" =~ '^[0-9]+\.[0-9]+\.[0-9]+$' ]]; then
        errore "La risposta di GitHub non contiene una versione valida."
        return 1
    fi
    campo "Versione installata" "$MANAGER_BUILD"
    campo "Ultima stabile" "$ultima"
    if [[ "$ultima" == "$MANAGER_BUILD" ]] || ! versioneminore "$MANAGER_BUILD" "$ultima"; then
        ok "Il manager e aggiornato."
        return 0
    fi
    print
    avviso "E disponibile una nuova versione stabile."
    nota "Per sicurezza il manager non esegue file di aggiornamento scaricati automaticamente."
    domanda "Aprire la release ufficiale su GitHub [s/N]:"
    IFS= read -r risposta || return 0
    case "${(L)risposta}" in
        (s|si|y|yes) open 'https://github.com/scarcellagb/robloxstudiomanager/releases/latest' >/dev/null 2>&1 ;;
        (*) ;;
    esac
}


menumanutenzione() {
    local scelta
    while true; do
        titolo
        sezione "Manutenzione"
        voce 1 "Pulizia"
        voce 2 "Riordina cartelle e archivi"
        voce 0 "Torna indietro"
        print
        domanda "Scelta:"
        IFS= read -r scelta || return 0
        case "$scelta" in
            (1) menupulizia ;;
            (2) titolo; sezione "Riordino"; print; spinneravvia "Riordino in corso"; ordina; spinnerferma; pausa ;;
            (0) return 0 ;;
            (*) ;;
        esac
    done
}

controllointegrita() {
    local fs fb meta stato_self stato_backup stato_meta
    titolo
    sezione "Integrita"
    spinneravvia "Verifico i file essenziali"
    fs="$(firmafile "$self" 2>/dev/null)"
    fb="$(firmafile "$backupcomando" 2>/dev/null)"
    meta="$(awk -F '\t' 'NR==1 {print $1 "|" $2 "|" $3}' "$integritafile" 2>/dev/null)"
    [[ "$fs" == "$MANAGER_FIRMA" ]] && stato_self="valida" || stato_self="non valida"
    [[ "$fb" == "$MANAGER_FIRMA" ]] && stato_backup="valido" || stato_backup="non valido"
    [[ "$meta" == "$MANAGER_BUILD|$MANAGER_DATA|$MANAGER_FIRMA" ]] && stato_meta="valido" || stato_meta="non valido"
    spinnerferma
    campo "Comando principale" "$stato_self"
    campo "Copia di recupero" "$stato_backup"
    campo "Metadati integrita" "$stato_meta"
    campo "Firma" "$MANAGER_FIRMA"
    if [[ "$stato_self" == "valida" && "$stato_backup" == "valido" && "$stato_meta" == "valido" ]]; then
        print
        ok "I file essenziali corrispondono alla firma prevista."
    else
        print
        errore "Uno o piu file essenziali non corrispondono alla firma prevista."
        nota "Riavvia il manager per tentare il ripristino automatico oppure reinstalla la release ufficiale."
    fi
}

mostrapercorsi() {
    titolo
    sezione "Percorsi"
    campo "Base" "$base"
    campo "Versioni" "$versioni"
    campo "Archivio" "$archivio"
    campo "Strumenti" "$strumenti"
    campo "Temporanei" "$temporanei"
    campo "Log Roblox" "$logroot"
}

mostrapermessi() {
    local fd au ac scelta
    while true; do
        titolo
        sezione "Permessi"
        if permessodownload; then fd="abilitato"; else fd="mancante"; fi
        if permessoautomazione; then au="abilitata"; else au="mancante"; fi
        if [[ "$(accessibilita)" == "si" ]]; then ac="abilitata"; else ac="mancante"; fi
        campo "Download" "$fd"
        campo "Automazione" "$au"
        campo "Accessibilita" "$ac"
        print
        voce 1 "Richiedi i permessi mancanti"
        voce 0 "Torna indietro"
        print
        domanda "Scelta:"
        IFS= read -r scelta || return 0
        case "$scelta" in
            (1) richiedipermessi ;;
            (0) return 0 ;;
            (*) ;;
        esac
    done
}

menuretecache() {
    local scelta rete n
    while true; do
        titolo
        sezione "Rete e cache"
        spinneravvia "Controllo la connessione"
        if reteattiva; then rete="raggiungibile"; else rete="non raggiungibile"; fi
        n="$(awk -F '\t' 'NF>=3 {n++} END {print n+0}' "$webcache" 2>/dev/null)"
        spinnerferma
        campo "Server Roblox" "$rete"
        campo "Voci cache CDN" "$n"
        campo "Durata cache CDN" "12 ore"
        print
        voce 1 "Svuota cache CDN"
        voce 0 "Torna indietro"
        print
        domanda "Scelta:"
        IFS= read -r scelta || return 0
        case "$scelta" in
            (1)
                : > "$webcache"
                ok "Cache CDN svuotata."
                pausa
                ;;
            (0) return 0 ;;
            (*) ;;
        esac
    done
}

mostraprivacy() {
    titolo
    sezione "Privacy e dati"
    campo "Catalogo" "locale"
    campo "Stato verifiche" "locale"
    campo "Compatibilita" "locale"
    campo "Cache CDN" "locale"
    campo "Credenziali" "non memorizzate"
    campo "Telemetria manager" "nessuna"
    print
    nota "I dati runtime restano nella cartella Strumenti e sono esclusi dal repository Git."
}

menuimpostazioni() {
    local scelta
    while true; do
        titolo
        sezione "Impostazioni"
        voce 1 "Percorsi"
        voce 2 "Permessi"
        voce 3 "Rete e cache"
        voce 4 "Privacy e dati"
        voce 0 "Torna indietro"
        print
        domanda "Scelta:"
        IFS= read -r scelta || return 0
        case "$scelta" in
            (1) mostrapercorsi; pausa ;;
            (2) mostrapermessi ;;
            (3) menuretecache ;;
            (4) mostraprivacy; pausa ;;
            (0) return 0 ;;
            (*) ;;
        esac
    done
}

menustrumenti() {
    local scelta
    while true; do
        titolo
        sezione "Strumenti"
        voce 1 "Diagnostica"
        voce 2 "Manutenzione"
        voce 3 "Integrita"
        voce 4 "Aggiornamenti"
        voce 0 "Torna indietro"
        print
        domanda "Scelta:"
        IFS= read -r scelta || return 0
        case "$scelta" in
            (1) diagnostica; pausa ;;
            (2) menumanutenzione ;;
            (3) controllointegrita; pausa ;;
            (4) titolo; controllaaggiornamento; pausa ;;
            (0) return 0 ;;
            (*) ;;
        esac
    done
}

diagnostica() {
    local rete acc corrente n
    titolo
    sezione "Diagnostica"
    campo "Manager" "$MANAGER_BUILD - $MANAGER_DATA"
    campo "Cartella base" "$base"
    campo "macOS" "$SISTEMA"
    campo "Processore" "$ARCHMAC"
    campo "Larghezza finestra" "$LARG colonne"
    spinneravvia "Controllo connessione e permessi"
    if reteattiva; then
        rete="raggiungibile"
    else
        rete="non raggiungibile"
    fi
    acc="$(accessibilita)"
    spinnerferma
    campo "Server Roblox" "$rete"
    if [[ "$acc" == "si" ]]; then
        campo "Accessibilita" "abilitata"
    else
        campo "Accessibilita" "non abilitata"
    fi
    corrente="$(awk -F '\t' 'NR==1 {print $1}' "$correnteweb" 2>/dev/null)"
    [[ "$corrente" == 0.* ]] && campo "Versione live" "$corrente"
    n="$(wc -l < "$catalogo" 2>/dev/null | tr -d ' ')"
    campo "Build in catalogo" "${n:-0}"
    campo "Rilevamento" "Roblox ufficiale + hash risolti + test locale"
    campo "Integrita" "SHA-256 locale e ripristino runtime"
    campo "Firma applicazioni" "Roblox 2CFABCH843 obbligatoria"
    campo "Validita test verde" "12 ore"
    elencoapp
    campo "Versioni installate" "${#APPS}"
    n="$(awk -F '\t' '$3=="supportata"' "$stato" 2>/dev/null | wc -l | tr -d ' ')"
    campo "Verificate valide" "${n:-0}"
    n="$(awk -F '\t' '$3=="obsoleta"' "$stato" 2>/dev/null | wc -l | tr -d ' ')"
    campo "Verificate out of date" "${n:-0}"
    if [[ "$acc" != "si" ]]; then
        print
        avviso "Senza il permesso di Accessibilita le verifiche restano incerte."
        info "Impostazioni di Sistema, Privacy e Sicurezza, Accessibilita, abilita Terminale."
    fi
}

adattafinestra
aggiornalayout
assicurasistema
richiedipermessi
titolo
sezione "Preparazione"
spinneravvia "Controllo cartelle e dati"
puliscistrumenti
pulisciprogetto
puliscitemporanei
preparadati
spinnerferma
avviaguardia

while true; do
    titolo
    print
    voce 1 "Versioni"
    voce 2 "Installate"
    voce 3 "Archivio"
    voce 4 "Strumenti"
    voce 5 "Impostazioni"
    voce 0 "Esci"
    print
    nota "Ogni funzione e raccolta nella propria area per mantenere il menu essenziale e non ripetitivo."
    print
    domanda "Scelta:"
    IFS= read -r scelta || exit 0
    case "$scelta" in
        (1) menuversioni ;;
        (2) menuinstallate ;;
        (3) menuarchivio ;;
        (4) menustrumenti ;;
        (5) menuimpostazioni ;;
        (0) exit 0 ;;
        (*) ;;
    esac
done
