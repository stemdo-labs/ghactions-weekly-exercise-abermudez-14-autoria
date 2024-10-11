## Ejercicio Semanal -- Alejandro Bermúdez


### Diagrama del flujo de trabajo

![imagen](https://github.com/user-attachments/assets/aa0d692e-2d4a-4b4c-a95d-8d6eebc36cf9)

---
---

### Archivos


#### Workflow_principal.yml

```yml

name: Workflow principal para llamar a los reusables 

on:
  push:
    branches:
      - main
      - development 
jobs:
  setear_entorno:
    runs-on: ubuntu-latest
    outputs:
      entorno: ${{ steps.entorno.outputs.entorno  }}
    steps:
          - name: Settear entorno
            id: entorno
            run: |
              if [ "${{ github.ref }}" == "refs/heads/main" ]; then
                 echo "Entorno configurado en Prod"
                 echo "entorno=prod" >> $GITHUB_OUTPUT
              elif [ "${{ github.ref }}" == "refs/heads/development" ]; then
                echo "Entorno configurado en UAT"
                echo "entorno=uat" >> $GITHUB_OUTPUT
              else
                echo "Rama distinta a main / develop, entorno no configurado"
              fi
  
  lanzar_CI:
      needs: setear_entorno
      uses: ./.github/workflows/CI_workflow.yml
      secrets: inherit
      with:
        entorno: ${{ needs.setear_entorno.outputs.entorno  }}

          
  lanzar_CD:
      needs: 
        - lanzar_CI
        - setear_entorno
      uses: ./.github/workflows/CD_workflow.yml
      secrets: inherit
      with:
        nombre_completo: ${{ needs.lanzar_CI.outputs.nombre_completo}}
        entorno: ${{ needs.setear_entorno.outputs.entorno  }}

            
            

```

----

#### CI_worfklow.yml

```yml

name: CI workflow

on:
  workflow_call:
    inputs:
      entorno:
        required: true
        type: string
        
    outputs:
      nombre_completo:
        value: ${{jobs.buildear.outputs.nombre_completo}}

jobs:



  testear:
      runs-on: ubuntu-latest
      if: ${{ inputs.entorno  == 'prod' }}
      steps:
          - name: test
            run: | 
              echo "Haciendo tests de cobertura de código..."
              sleep 5

  buildear:
    if: ${{ always() }}
    needs: testear
    runs-on: ubuntu-latest
    outputs:
      nombre_completo: ${{steps.tag-imagen.outputs.nombre_completo}}
    environment: ${{inputs.entorno}}
    steps:
    - name: Clonar repo
      uses: actions/checkout@v3
      

  
    - name: Leer la versión del package.json
      id: coger_version
      run: |
           version=$(jq -r .version package.json)
           echo "version=$version" >> $GITHUB_OUTPUT
      shell: bash

    - name: Taggear con custom action
      id: tag-imagen
      uses: ./.github/actions/  
      with:
        nombre_imagen: 'angular_ab'
        version: "${{ steps.coger_version.outputs.version }}"

    - name: Construir imagen
      run: docker build -t ${{ vars.USUARIO_DOCKER }}/${{steps.tag-imagen.outputs.nombre_completo}} .




    
    - name: Hacer login
      uses: docker/login-action@v3
      with:
        username: ${{ vars.USUARIO_DOCKER }}
        password: ${{ secrets.PASSWORD_DOCKER }}

    - name: Verificar nombre completo 
      run: echo "Nombre completo --> ${{ steps.tag-imagen.outputs.nombre_completo }}"


    - name: Subir imagen al registry
      run: docker push ${{ vars.USUARIO_DOCKER }}/${{steps.tag-imagen.outputs.nombre_completo}}

```

----

#### CD_workflow.yml

```yml

name: CD workflow

on:
  workflow_call:
    inputs:
      nombre_completo:
        required: true
        type: string
      entorno:
        required: true
        type: string
   
jobs:
  desplegar_CD:
    runs-on: ubuntu-latest
    environment: ${{inputs.entorno}}
    steps:


    - name: Bajar imagen del registry
      run: docker pull ${{ vars.USUARIO_DOCKER }}/${{ inputs.nombre_completo }} 

    - name: Runnear el contenedor
      run: |
       docker run -d -p 80:8080 --name angular_ab ${{ vars.USUARIO_DOCKER }}/${{ inputs.nombre_completo }}
        
    - name: Hacer curl al nginx
      run: curl http://localhost:80
    

```

----

#### Action.yml

```yaml

name: Nombre completo
description: Etiquetar imagen

inputs:
  nombre_imagen:
    required: true
  version:
    required: true

outputs:
  nombre_completo:
    value: ${{steps.nombre_completo.outputs.nombre_completo}}

runs:
  using: "composite"
  steps:
      
    - name: Guardar nombre completo 
      id: nombre_completo
      run: echo "nombre_completo=${{ inputs.nombre_imagen }}:${{ inputs.version }}" >> $GITHUB_OUTPUT
      shell: bash



```

----
----

### Configuraciones

#### Configuración de entornos y variables

- Entornos
  
  ![imagen](https://github.com/user-attachments/assets/8e4a5ef2-dd3b-4455-a237-c26e5a01e6df)

   Dentro de cada entorno, se encuentran el secreto`PASSWORD_DOCKER` y la variable `USUARIO_DOCKER`

  ![imagen](https://github.com/user-attachments/assets/40f7a95e-b09b-49f4-b211-3fbeea4a8676)

----

#### Aprobadores de código

##### Archivo CODEOWNERS
`* @abermudez-14`


- Crear regla para apuntar al archivo `CODEOWNERS`
  
![imagen](https://github.com/user-attachments/assets/d3af70cb-4e40-48f4-86b8-a132a3f2d023)

