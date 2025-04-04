cmake_minimum_required(VERSION 3.10)
project(StudentEvaluationProject)

set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)


# Recherche de l'exécutable lcov
find_program(LCOV_EXEC lcov)
if(NOT LCOV_EXEC)
    message(FATAL_ERROR "lcov n'a pas été trouvé. Veuillez l'installer pour générer le rapport de couverture.")
endif()

# Recherche de l'exécutable genhtml
find_program(GENHTML_EXEC genhtml)
if(NOT GENHTML_EXEC)
    message(FATAL_ERROR "genhtml n'a pas été trouvé. Veuillez l'installer pour générer le rapport de couverture.")
endif()


# Rechercher l'exécutable cppcheck
find_program(CPPCHECK_EXECUTABLE NAMES cppcheck)
if(CPPCHECK_EXECUTABLE)
    message(STATUS "Found cppcheck: ${CPPCHECK_EXECUTABLE}")

    # Créer une cible personnalisée pour lancer cppcheck et générer un rapport XML.
    # Notez que cppcheck écrit le rapport XML sur stderr.
    add_custom_target(run_cppcheck
        COMMAND ${CPPCHECK_EXECUTABLE} 
                --xml 
                --xml-version=2 
                --enable=all 
                --inconclusive 
                --std=c++11 
                --suppress=missingIncludeSystem 
                --suppress=unusedFunction 
                ${CMAKE_SOURCE_DIR}/src 2> ${CMAKE_BINARY_DIR}/cppcheck.xml
        WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
        COMMENT "Running cppcheck static analysis and generating XML report..."
    )
else()
    message(WARNING "cppcheck n'a pas été trouvé. Veuillez l'installer.")
endif()




#Activer les tests pour la commande ctest
enable_testing()

# Création de la bibliothèque à partir du code source
add_library(student_evaluation src/student_evaluation.cpp)


# Intégration de Google Test via FetchContent
include(FetchContent)
FetchContent_Declare(
  googletest
  URL https://github.com/google/googletest/archive/release-1.11.0.zip
)
# Pour Windows, afin d'assurer la cohérence des réglages CRT
set(gtest_force_shared_crt ON CACHE BOOL "" FORCE)
FetchContent_MakeAvailable(googletest)

# Ajout de l'exécutable de tests
add_executable(test_student_evaluation tests/test_student_evaluation.cpp)
target_link_libraries(test_student_evaluation 
                      student_evaluation gtest_main
                      )

option(CODE_COVERAGE "Enable coverage reporting" ON)
    if(CODE_COVERAGE)
        if(CMAKE_CXX_COMPILER_ID MATCHES "GNU|Clang")
            message(STATUS "Code coverage enabled")
            # Pour faciliter l'analyse, désactiver les optimisations et activer le profilage
            target_compile_options(test_student_evaluation PRIVATE -O0 -fprofile-arcs -ftest-coverage -g)
            target_link_options( test_student_evaluation PRIVATE -fprofile-arcs -ftest-coverage)
            
        else()
            message(WARNING "Code coverage is not supported for this compiler")
        endif()
    endif()



# Détection automatique des tests
include(GoogleTest)
gtest_discover_tests(test_student_evaluation)

# Pour générer un rapport xml de test compatible jenkins
add_test(
    NAME TestStudentEvaluation
    COMMAND test_student_evaluation --gtest_output=xml:${CMAKE_BINARY_DIR}/report.xml
)



# Cible personnalisée pour générer le rapport de couverture
# Cible pour générer le rapport de couverture (utilise lcov et genhtml)
add_custom_target(coverage
    COMMAND lcov --rc lcov_branch_coverage=1 --directory "/home/testeur/Projets/testingincpp/build" --capture --output-file coverage.info
    COMMAND lcov --extract coverage.info "/home/testeur/Projets/testingincpp/src/*" --output-file coverage-src.info
    COMMAND genhtml coverage-src.info --output-directory coverage_report
    WORKING_DIRECTORY ${CMAKE_BINARY_DIR}
    COMMENT "Generating code coverage report..."
)

