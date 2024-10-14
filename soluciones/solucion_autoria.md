## Solución -- Alejandro Bermúdez

### Modificación del `Action.yml`

- Se modifica el step `guardar nombre completo` de la custom action para que, dependiendo en la rama en la que se haya hecho el push, concatenar solo el nombre de la imagen y la versión o añadirle `-snapshot`.
  

```yml

    - name: Guardar nombre completo 
      id: nombre_completo
      run: |
        if [ "${{ github.ref }}" == "refs/heads/main" ]; then
           echo "nombre_completo=${{ inputs.nombre_imagen }}:${{ inputs.version }}" >> $GITHUB_OUTPUT
        elif [ "${{ github.ref }}" == "refs/heads/development" ]; then
         echo "nombre_completo=${{ inputs.nombre_imagen }}:${{ inputs.version }}-snapshot" >> $GITHUB_OUTPUT
        else
          echo "Rama distinta a main / develop "
        fi

```
---
---


### Pruebas

- Hacemos el push a `main`

![imagen](https://github.com/user-attachments/assets/6e323d8b-bd16-4310-b643-1a611e0b4f76)


---
---


- Hacemos el push a `development`

![imagen](https://github.com/user-attachments/assets/8b3c35c0-f4db-4a28-80e1-49413fb85478)




### Versiones subidas al registry

![imagen](https://github.com/user-attachments/assets/00463af2-f924-4d44-8ed9-a2a914ddc8d0)

