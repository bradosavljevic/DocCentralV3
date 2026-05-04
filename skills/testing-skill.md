# Skill: Testing

## Obavezno testirati

- unos dokumenta sa sledećim brojem
- unos dokumenta sa rezervisanim brojem
- concurrent zavođenje
- validacije
- approval sekvencijalno
- approval na grupu
- odbijanje
- podsetnike
- arhiviranje
- zaključenje godine
- zaključanu godinu
- brisanje/deaktivaciju partnera
- backend permissions
- Power Automate greške
- audit log

## Pravilo

Svaka poslovna funkcija mora imati happy path, validation path, permission path i error path.


## Naming / postojeći objekti

Koristi solution `DocCentralV3`, connection references `CR_DocCentralV3_*`, environment variables `EV_DocCentralV3_*` i cloud flow nazive sa prefixom `CF_DocCentralV3_`. Ne koristiti prefix `PA_` za nove flow-ove. Svi zasebni code fajlovi moraju biti u folderu `PACode`.
