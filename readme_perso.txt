build docker container: 
docker build -t bc/cert-issuer:1.0 .

If modif done, create new image :
docker ps -l
docker commit <container for your bc/cert-issuer> my_cert_issuer
puis utiliser cette image : docker run -it my_cert_issuer bash (sinon image de base, fresh start : docker run -it bc/cert-issuer:1.0 bash)

Créer des wallets, adresses etc depuis bitcoin-cli (regtest : bitcoind -regtest -daemon -conf=/root/.bitcoin/bitcoin.conf) -6> la conf pour pas devoir rentrer le password etc a chaqeu fois (foo et bar)
Et de meme bitcoin-cli on lui passe le .conf en chemin bitcoin-cli -regtest -conf=/root/.bitcoin/bitcoin.conf listwallets
Attention parfois j'alias bitcoin cli en bitcoin-cli -regtest -conf=filepath

(Faut d'abord kill les process non-regtest : px aux --> kill -9 PID)


#L'ACCES AU PRIV KEY VIA BITCOIN-CLI SUR DOCKER EST DEPRECIE.
SOlution : 
# 0) créer wallet descriptor si besoin
bitcoin-cli -regtest createwallet issuer_wallet true

# 1) obtenir WIF (ici on part du principe que tu as une WIF dans $WIF) --> Utiliser un script python pour générer une prv key format WIF et le foutre en var
WIF="cT..."   # ou récupère avec script hex2wif.py

# 2) form descriptor (wrapped segwit) -->
DESCR="sh(wpkh($WIF))"

# 3) get descriptor checked
DESC_WITH_CHECKSUM=$(bitcoin-cli -regtest getdescriptorinfo "$DESCR" | grep -oP '"descriptor":\s*"\K[^"]+') --> On grep pas, on prendra plutot direct le desc présent dans $DESCINFO

# 4) import
bitcoin-cli -regtest -rpcwallet=issuer_wallet importdescriptors '[{"desc":"'"$DESC_WITH_CHECKSUM"'","internal":false,"label":"issuer","timestamp":"now"}]'

# 5) get address
bitcoin-cli -regtest -rpcwallet=issuer_wallet deriveaddresses "$DESC_WITH_CHECKSUM"
--> Apres tout ca, la private est bien enregistrée (/root/regtest/wallets/$wallet_name)
--> En prod si on génère des wallets pour les bénéficiares (ou les issuers d'ailleurs) faudra utiliser tout ce cheminement (sans regtest bien sur, et peut etre sur une autre blockchain)

Je stocke un volume de conteneur ici : C:\Users\rodgo\bitcoin_data (contient wallet)
pour lancer un conteneur avec ce volume : docker run -it -v C:\Users\rodgo\bitcoin_data:/root/.bitcoin my_cert_issuer bash
pour écraser a la fin d'une cession (save & ecrase) : docker cp <container_id>:/root/.bitcoin C:\Users\rodgo\bitcoin_data


Créer des certifs avec cert- (voir repo cert-tools)

conf.ini doit etre configuré ainsi que pk_issuer
Puis quand tout est ready : cert-issuer -c /etc/cert-issuer/conf.ini