# computer-graphics
- Documentation DRAGON à l’aide de l’IA
    
    ---
    
    ### 1. **Les bibliothèques importées :**
    
    ```cpp
    #include <GL/glew.h>     // Gestion des extensions OpenGL
    #include <GLFW/glfw3.h>  // Gestion des fenêtres et événements
    #include <glm/glm.hpp>   // Bibliothèque mathématique pour OpenGL (matrices et vecteurs)
    #include <glm/gtc/matrix_transform.hpp>  // Fonctions de transformation (rotation, translation)
    #include <glm/gtc/type_ptr.hpp>  // Conversion des matrices et vecteurs pour OpenGL
    #include <iostream>      // Gestion de l’entrée/sortie standard
    
    ```
    
    Ces bibliothèques sont nécessaires pour :
    
    - **GLEW** : Permet d'utiliser les extensions OpenGL non incluses dans le standard.
    - **GLFW** : Gère la création de fenêtres, les événements clavier/souris et le contexte OpenGL.
    - **GLM** : Bibliothèque mathématique pour la manipulation des matrices et des transformations (indispensable pour la 3D).
    
    ### 2. **Chargement des données du modèle Dragon :**
    
    ```cpp
    #include "DragonData.h" // Données du Dragon de Stanford (sommets, indices)
    
    ```
    
    Le fichier **`DragonData.h`** contient les **sommets** et **indices** du modèle 3D du Dragon de Stanford. Chaque sommet contient :
    
    - **Position** (X, Y, Z)
    - **Normale** (NX, NY, NZ) : utilisée pour l'éclairage et les effets de lumière.
    - **Coordonnées de texture** (pas utilisées ici, mais utiles pour appliquer des textures).
    
    ---
    
    ### 3. **Shaders**
    
    Les **shaders** sont des petits programmes exécutés directement sur le GPU, permettant de gérer les transformations des vertices et l'éclairage.
    
    - **Vertex Shader** :
        - Transforme la position des sommets d'un modèle selon les matrices de **modèle**, **vue** (caméra), et **projection** (perspective).
        - Passe les normales pour l'éclairage.
    
    ```cpp
    std::string vertexShaderSource = R"(
    #version 330 core
    layout(location = 0) in vec3 position; // Position des sommets
    layout(location = 1) in vec3 normal;   // Normales des sommets
    out vec3 v_Normal; // Passer la normale au fragment shader
    uniform mat4 model;     // Matrice de transformation du modèle
    uniform mat4 projection; // Matrice de projection (perspective)
    uniform mat4 view;      // Matrice de la caméra (vue)
    
    void main() {
        gl_Position = projection * view * model * vec4(position, 1.0); // Transformation des positions
        v_Normal = normal; // Transférer la normale
    })";
    
    ```
    
    - **Fragment Shader** :
        - Calcule la couleur finale de chaque fragment (pixel), en utilisant l’éclairage avec la loi de Lambert.
    
    ```cpp
    std::string fragmentShaderSource = R"(
    #version 330 core
    in vec3 v_Normal; // Normale reçue du vertex shader
    out vec4 color;   // Couleur finale du fragment
    const vec3 LightDirection = vec3(1.0, -1.0, -1.0); // Direction de la lumière
    
    void main() {
        vec3 N = normalize(v_Normal); // Normaliser la normale
        vec3 L = normalize(-LightDirection); // Calculer la direction de la lumière
        float LambertDiffuse = max(dot(N, L), 0.0); // Loi de Lambert pour l'éclairage diffus
        color = vec4(vec3(LambertDiffuse), 1.0); // Couleur finale du fragment
    })";
    
    ```
    
    ### 4. **Compilation des shaders :**
    
    ```cpp
    GLuint CompileShader(const std::string &source, GLenum type) {
        GLuint shader = glCreateShader(type); // Créer un shader (vertex ou fragment)
        const char *src = source.c_str();
        glShaderSource(shader, 1, &src, nullptr);
        glCompileShader(shader); // Compiler le shader
    }
    
    ```
    
    Les shaders sont ensuite **liés** dans un programme :
    
    ```cpp
    void CreateShaderProgram() {
        GLuint vertexShader = CompileShader(vertexShaderSource, GL_VERTEX_SHADER);
        GLuint fragmentShader = CompileShader(fragmentShaderSource, GL_FRAGMENT_SHADER);
    
        shaderProgram = glCreateProgram();
        glAttachShader(shaderProgram, vertexShader);  // Attacher le vertex shader
        glAttachShader(shaderProgram, fragmentShader); // Attacher le fragment shader
        glLinkProgram(shaderProgram); // Lier le programme
    }
    
    ```
    
    ---
    
    ### 5. **Buffers et Attributs de Vertex**
    
    Lors de l'initialisation, on configure deux **buffers** :
    
    - **VBO (Vertex Buffer Object)** : Stocke les sommets du dragon.
    - **EBO (Element Buffer Object)** : Stocke les indices qui définissent comment les sommets sont connectés pour former des triangles.
    
    ```cpp
    void Init() {
        // Créer et lier les buffers
        glGenBuffers(1, &vbo);
        glGenBuffers(1, &ebo);
    
        // Transférer les données des sommets au VBO
        glBindBuffer(GL_ARRAY_BUFFER, vbo);
        glBufferData(GL_ARRAY_BUFFER, sizeof(DragonVertices), DragonVertices, GL_STATIC_DRAW);
    
        // Transférer les données des indices à l’EBO
        glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, ebo);
        glBufferData(GL_ELEMENT_ARRAY_BUFFER, sizeof(DragonIndices), DragonIndices, GL_STATIC_DRAW);
    
        // Définir les attributs de vertex (position et normales)
        glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 8 * sizeof(float), (void*)0);
        glEnableVertexAttribArray(0);
        glVertexAttribPointer(1, 3, GL_FLOAT, GL_FALSE, 8 * sizeof(float), (void*)(3 * sizeof(float)));
        glEnableVertexAttribArray(1);
    }
    
    ```
    
    ### 6. **Transformation et Rendu :**
    
    Le modèle est ensuite rendu dans une boucle continue où les matrices de transformation sont appliquées et le dragon est dessiné avec **GL_TRIANGLES**.
    
    - **Matrice de modèle** : Applique une **rotation** autour de l'axe Y.
    - **Matrice de vue** : Reculer la caméra.
    - **Matrice de projection** : Applique la projection en perspective.
    
    ```cpp
    glm::mat4 model = glm::rotate(glm::mat4(1.0f), glm::radians(angleY), glm::vec3(0.0f, 1.0f, 0.0f));
    glm::mat4 view = glm::translate(glm::mat4(1.0f), glm::vec3(0.0f, 0.0f, -20.0f));
    glm::mat4 projection = glm::perspective(glm::radians(60.0f), 800.0f / 600.0f, 0.1f, 200.0f);
    
    // Passer les matrices au shader
    glUniformMatrix4fv(glGetUniformLocation(shaderProgram, "model"), 1, GL_FALSE, glm::value_ptr(model));
    glUniformMatrix4fv(glGetUniformLocation(shaderProgram, "projection"), 1, GL_FALSE, glm::value_ptr(projection));
    glUniformMatrix4fv(glGetUniformLocation(shaderProgram, "view"), 1, GL_FALSE, glm::value_ptr(view));
    
    ```
    
    Le dragon est dessiné :
    
    ```cpp
    glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, ebo);
    glDrawElements(GL_TRIANGLES, sizeof(DragonIndices) / sizeof(unsigned short), GL_UNSIGNED_SHORT, 0);
    
    ```
    
    ### 7. **Boucle principale et gestion des événements :**
    
    Dans la boucle principale, le programme attend les événements utilisateur et redessine le dragon continuellement :
    
    ```cpp
    while (!glfwWindowShouldClose(window)) {
        Render(window); // Dessine le dragon
        glfwPollEvents(); // Gère les entrées utilisateur
    }
    
    ```
    
    ---