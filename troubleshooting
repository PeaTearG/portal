#!/bin/bash

load_sessions() {
    local dump_output
    dump_output=$(mktemp)
    sudo cz-memcachedump -j > "$dump_output"
    declare -gA sessions=()
    while IFS= read -r entry; do
        key=$(echo "$entry" | jq -r '.[0]')
        # Only process keys with a length of 10
        if [[ ${#key} -eq 10 ]]; then
            sessiondetails=$(echo $entry | jq -r '.[1]')
            client_id=$(echo $sessiondetails | jq .clientId)
            blockLocalDnsRequests=$(echo $sessiondetails | jq .blockLocalDnsRequests)
            distinguishedName=$(echo $sessiondetails | jq .distinguishedName)
            dnsSearchDomains=$(echo $sessiondetails | jq .dnsSearchDomains)
            dnsServers=$(echo $sessiondetails | jq .dnsServers)
            loggedIn=$(echo $sessiondetails | jq .loggedIn)
            username=$(echo "$distinguishedName" | awk -F ',' '{print $2}' | xargs)
            username="${username:3}"
            if [[ "$client_id" =~ ^[0-9]+$ && -n "$username" ]]; then
                sessions["$username"]="$client_id"
            fi
        fi
    done < <(jq -c 'to_entries | .[] | [(.key), .value]' "$dump_output")

    rm -f "$dump_output"
}

# Menu loop
while true; do
    load_sessions

    if [ ${#sessions[@]} -eq 0 ]; then
        dialog --msgbox "No sessions found." 8 40
        exit 1
    fi

    OPTIONS=()
    for username in "${!sessions[@]}"; do
        OPTIONS+=("$username" "")
    done
    OPTIONS+=("__RELOAD__" "Reload the session list")

    tempfile=$(mktemp)
    dialog --clear \
           --backtitle "Session Selector" \
           --title "Select a session" \
           --menu "Choose a username (or reload):" 20 70 12 \
           "${OPTIONS[@]}" 2>"$tempfile"

    selected=$(<"$tempfile")
    rm -f "$tempfile"

    if [ -z "$selected" ]; then
        echo "Cancelled."
        clear && exit 0
    elif [ "$selected" == "__RELOAD__" ]; then
        continue  # Loop again, reload sessions
    fi

    sessionID=${sessions["$selected"]}
    clear && sudo ip netns exec "portal$sessionID" bash

    break
done
