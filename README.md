# Boîte à outils de vérification mobile (MVT)
# Copyright (c) 2021 Développeurs de projets MVT.
# Voir le fichier 'LICENSE' pour les autorisations d'utilisation et de copie, ou trouvez une copie sur
# https://github.com/mvt-project/mvt/blob/main/LICENSE

importer le système d'  exploitation
importer  json
importer la  journalisation
importer  pkg_resources
de  tqdm  importer  tqdm

de  mvt . commun . utils  importe  get_sha256_from_file_path
de . module . adb . importation de base  AndroidExtraction 

log  =  journalisation . getLogger ( __nom__ )

# A FAIRE : vaudrait mieux remplacer tqdm par rich.progress pour réduire
# le nombre de dépendances. Besoin d'enquêter pour savoir si
# il est possible d'avoir un système de rappel simialr.
classe  PullProgress ( tqdm ):
    """PullProgress est un système de mise à jour tqdm pour les téléchargements d'APK."""

    def  update_to ( self , file_name , current , total ):
        si le  total  n'est  pas  Aucun :
            soi . total  =  total
        soi . update ( current  -  self . n )


 Forfait classe :
    """Package indique un nom de package et tous les fichiers qui lui sont associés."""

    def  __init__ ( self , name , files = None ):
        soi . nom  =  nom
        soi . fichiers  =  fichiers  ou []


classe  DownloadAPK ( AndroidExtraction ):
    """DownloadAPKs est la classe principale exploitant le téléchargement d'APK
    de l'appareil."""

    def  __init__ ( self , output_folder = None , all_apks = False , packages = None ):
        """Initialiser le module.
        :param output_folder : chemin vers le dossier où les données doivent être stockées
        :param all_apks : booléen indiquant s'il faut télécharger tous les packages
                         ou filtrer les produits connus 
        :param packages : liste de packages fournie, généralement pour les vérifications JSON
        """
        super (). __init__ ( file_path = None , base_folder = None ,
                         dossier_sortie = dossier_sortie )

        soi . output_folder_apk  =  Aucun
        soi . packages  =  packages  ou []
        soi . all_apks  =  all_apks

        soi . _safe_packages  = []

    @ méthode de classe
    def  from_json ( cls , json_path ):
        """ Initialisez cette classe à partir d'un fichier packages.json existant.
        :param json_path : chemin d'accès au fichier packages.json à analyser.
        """
        avec  open ( json_path , "r" ) comme  handle :
            données  =  json . charge ( poignée )

            paquets  = []
            pour l'  entrée  dans les  données :
                package  =  Package ( entrée [ "nom" ], entrée [ "fichiers" ])
                paquets . ajouter ( paquet )

            return  cls ( packages = packages )

    def  _load_safe_packages ( self ):
        """Charger les noms de packages connus.
        """
        safe_packages_path  =  os . chemin . join ( "données" , "safe_packages.txt" )
        safe_packages_string  =  pkg_resources . resource_string ( __name__ , safe_packages_path )
        safe_packages_list  =  safe_packages_string . décoder ( " utf-8 " ). diviser ( " \n " )
        soi . _safe_packages . étendre ( safe_packages_list )

    def  _clean_output ( self , output ):
        """ Nettoyer la sortie de la commande shell adb.
        :param output : sortie de la commande à nettoyer.
        """
         sortie de retour . bande (). remplacer ( "paquet:" , "" )

    def  get_packages ( self ):
        """Récupérez les noms de package de l'appareil à l'aide d'adb.
        """
        log . info ( "Récupération des noms de packages ..." )

        si  pas  auto . all_apks :
            soi . _load_safe_packages ()

        sortie  =  soi . _adb_command ( "pm list packages" )
        total  =  0
        pour la  ligne  de  sortie . diviser ( " \n " ):
            package_name  =  self . _clean_output ( ligne )
            si  nom_paquet  ==  "" :
                Continuez

            total  +=  1

            si  pas  auto . all_apks  et  package_name  dans  self . _safe_packages :
                Continuez

            si  package_name  n'est pas  dans  self . forfaits :
                soi . paquets . append ( Package ( package_name ))

        log . info ( "Il y a %d packages installés sur l'appareil. J'ai sélectionné %d pour inspection." ,
                 total , len ( self . packages ))

    def  pull_package_file ( self , package_name , remote_path ):
        """ Extrayez les fichiers liés à un package spécifique de l'appareil.
        :param package_name : Nom du package à télécharger
        :param remote_path : chemin vers le fichier à télécharger
        :returns : chemin vers la copie locale
        """
        log . info ( "Téléchargement de %s ..." , remote_path )

        nom_fichier  =  ""
        if  "==/"  dans  remote_path :
            nom_fichier  =  "_"  +  chemin_distant . split ( "==/" )[ 1 ]. remplacer ( ".apk" , "" )

        chemin_local  =  os . chemin . join ( self . output_folder_apk ,
                                  f" { package_name } { file_name } .apk" )
        name_counter  =  0
        tandis que  True :
            si  pas  os . chemin . existe ( chemin_local ):
                Pause

            name_counter  +=  1
            chemin_local  =  os . chemin . join ( self . output_folder_apk ,
                                      f" { package_name } { file_name } _ { name_counter } .apk" )

        essaie :
            avec  PullProgress ( unit = 'B' , unit_divisor = 1024 , unit_scale = True ,
                              miniters = 1 ) comme  pp :
                soi . _adb_download ( remote_path , local_path ,
                                   progress_callback = pp . update_to )
        sauf  Exception  comme  e :
            log . exception ( "Impossible d'extraire le fichier de package de %s : %s" ,
                          chemin_distant , e )
            soi . _adb_reconnect ()
            retour  Aucun

        retourner  local_path

    def  pull_packages ( self ):
        """Téléchargez tous les fichiers de tous les packages sélectionnés à partir de l'appareil.
        """
        log . info ( "Démarrage de l'extraction des fichiers APK installés dans le dossier %s" , self . output_folder )

        si  pas  os . chemin . existe ( self . dossier_sortie ):
            os . mkdir ( self . dossier_sortie )

        log . info ( "Téléchargement des packages depuis l'appareil. Cela peut prendre un certain temps ..." )

        soi . output_folder_apk  =  os . chemin . join ( self . output_folder , "apks" )
        si  pas  os . chemin . existe ( self . output_folder_apk ):
            os . mkdir ( self . output_folder_apk )

        total_packages  =  len ( self . packages )
        compteur  =  0
        pour le  paquet  en  soi . forfaits :
            compteur  +=  1

            log . info ( "[%d/%d] Package : %s" , counter , total_packages , package . name )

            essaie :
                sortie  =  soi . _adb_command ( f"pm path { package . name } " )
                sortie  =  soi . _clean_output ( sortie )
                si  pas  sortie :
                    Continuez
            sauf  Exception  comme  e :
                log . exception ( "Impossible d'obtenir le chemin du package %s : %s" , package . name , e )
                soi . _adb_reconnect ()
                Continuez

            # Parfois, le chemin du package contient plusieurs lignes pour plusieurs apks.
            # Nous parcourons chaque ligne et téléchargeons chaque fichier.
            pour le  chemin  en  sortie . diviser ( " \n " ):
                chemin_périphérique  =  chemin . bande ()
                chemin_fichier  =  soi . pull_package_file ( package . nom , chemin_périphérique )
                si  non  file_path :
                    Continuez

                # Nous ajoutons les métadonnées apk à l'objet package.
                paquet . fichiers . ajouter ({
                    "chemin" : chemin_périphérique ,
                    "nom_local" : chemin_fichier ,
                    "sha256" : get_sha256_from_file_path ( file_path ),
                })

    def  save_json ( self ):
        """Enregistrez les résultats dans le fichier package.json.
        """
        json_path  =  os . chemin . join ( self . output_folder , "packages.json" )
        paquets  = []
        pour le  paquet  en  soi . forfaits :
            paquets . ajouter ( package . __dict__ )

        avec  open ( json_path , "w" ) comme  handle :
            json . dump ( packages , handle , indent = 4 )

    def  run ( self ):
        """ Exécutez toutes les étapes de fetch-apk.
        """
        soi . _adb_connect ()
        soi . obtenir_paquets ()
        soi . pull_packages ()
        soi . save_json ()
        soi . _adb_disconnect ()
