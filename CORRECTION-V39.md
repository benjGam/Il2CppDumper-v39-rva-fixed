# Correction du décodage des données RVA — IL2CPP metadata V39

## Cause validée sur les fichiers transmis

Le champ 83939 de `<PrivateImplementationDetails>` porte les attributs IL2CPP `0x0133`, dont
`FIELD_ATTRIBUTE_HAS_FIELD_RVA` (`0x0100`). Son initialiseur est un bloc binaire brut,
commençant par `E1 2F FF 13 00 00 00 00`, à l'offset absolu `0x16F9860`
dans `global-metadata.dat`. Ce n'est pas un entier compressé ; `0xE1` n'est pas un
préfixe d'entier compressé valide.

L'analyse des entrées des valeurs par défaut de ce fichier a mis en évidence **2 078 champs HasFieldRVA**.
La précédente version du dumper tentait de les décoder comme des constantes, ce qui
pouvait produire des valeurs erronées (ou cette exception).

## Correction

- `Il2CppConstants.cs` : ajout du drapeau `FIELD_ATTRIBUTE_HAS_FIELD_RVA`.
- `Il2CppDecompiler.cs` : ces champs sont annotés `/* FieldRVA: raw data at metadata offset 0x... */`
  au lieu de recevoir une fausse constante.
- `DummyAssemblyGenerator.cs` : pas de décodage de constante pour ces champs ;
  un attribut `MetadataOffset` garde le pointeur dans `global-metadata.dat`.
- La lecture des constantes ordinaires reste inchangée. Le garde-fou contre
  des données corrompues, introduit dans la V2, reste en place.

**Limite :** les octets de l'initialiseur ne sont pas réinjectés dans le DummyDll
(`FieldDefinition.InitialValue` n'est pas renseigné), car déterminer leur taille
exacte exige un traitement supplémentaire du type/layout. Leur offset original
est conservé. Le correctif règle donc le décodage et les valeurs factices,
mais ne prétend pas reconstruire les données RVA dans les DLL exportées.

## Compilation

À partir du dossier `Il2CppDumper` :

```powershell
dotnet build -c Release -f net8.0
```

Le fichier source `Il2CppDumper/Libraries/Il2CppDummyDll.dll` est inclus
pour satisfaire la référence `Resource1.resx`.

**Validation :** analyse directe des tables metadata et flags des types dans
les fichiers d'origine. Compilation et exécution .NET non testées dans
l'environnement de génération (SDK .NET absent).
