package com.aconex.scrutineer;

import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.SQLException;

import com.aconex.scrutineer.elasticsearch.ElasticSearchDownloader;
import com.aconex.scrutineer.elasticsearch.ElasticSearchIdAndVersionStream;
import com.aconex.scrutineer.elasticsearch.ElasticSearchSorter;
import com.aconex.scrutineer.elasticsearch.ElasticSearchTransportClientFactory;
import com.aconex.scrutineer.elasticsearch.IdAndVersionDataReaderFactory;
import com.aconex.scrutineer.elasticsearch.IdAndVersionDataWriterFactory;
import com.aconex.scrutineer.elasticsearch.IteratorFactory;
import com.aconex.scrutineer.jdbc.JdbcIdAndVersionStream;
import com.beust.jcommander.JCommander;
import com.fasterxml.sort.DataReaderFactory;
import com.fasterxml.sort.DataWriterFactory;
import com.fasterxml.sort.SortConfig;
import com.fasterxml.sort.Sorter;
import com.fasterxml.sort.util.NaturalComparator;
import com.google.common.base.Function;
import org.apache.commons.lang3.SystemUtils;
import org.elasticsearch.client.transport.TransportClient;
import org.slf4j.Logger;

public class Scrutineer {

    private static final Logger LOG = LogUtils.loggerForThisClass();

    public static void main(String[] args) {
        try {
            execute(new Scrutineer(parseOptions(args)));
        } catch (Exception e) {
            LOG.error("Failure during Scrutineering", e);
            System.exit(1);
        }
    }

    static void execute(Scrutineer scrutineer) {
        try {
            scrutineer.verify();
        } finally {
            scrutineer.close();
        }
    }

    private static ScrutineerCommandLineOptions parseOptions(String[] args) {
        ScrutineerCommandLineOptions options = new ScrutineerCommandLineOptions();
        JCommander jCommander = new JCommander(options);
        jCommander.parse(args);
        return options;
    }

    public void verify() {
        Function<Long, Object> formatter = createFormatter();
        IdAndVersionStreamVerifierListener verifierListener = createVerifierListener(formatter);

        this.verify(verifierListener);
    }

    public void verify(IdAndVersionStreamVerifierListener verifierListener) {
        idAndVersionFactory = createIdAndVersionFactory();
        ElasticSearchIdAndVersionStream elasticSearchIdAndVersionStream = createElasticSearchIdAndVersionStream(options);
        JdbcIdAndVersionStream jdbcIdAndVersionStream = createJdbcIdAndVersionStream(options);

        verify(elasticSearchIdAndVersionStream, jdbcIdAndVersionStream, new IdAndVersionStreamVerifier(), verifierListener);
    }

    public void close() {
        closeJdbcConnection();
        closeElasticSearchConnections();
    }

    void closeElasticSearchConnections() {
        if (client != null) {
            client.close();
        }
    }

    void closeJdbcConnection() {
        try {
            if (connection != null) {
                connection.close();
            }
        } catch (SQLException e) {
            throw new RuntimeException(e);
        }
    }

    void verify(ElasticSearchIdAndVersionStream elasticSearchIdAndVersionStream, JdbcIdAndVersionStream jdbcIdAndVersionStream, IdAndVersionStreamVerifier idAndVersionStreamVerifier, IdAndVersionStreamVerifierListener verifierListener) {
        idAndVersionStreamVerifier.verify(jdbcIdAndVersionStream, elasticSearchIdAndVersionStream, verifierListener);
    }

    private Function<Long, Object> createFormatter() {
        Function<Long, Object> formatter = PrintStreamOutputVersionStreamVerifierListener.DEFAULT_FORMATTER;
        if (options.versionsAsTimestamps) {
            formatter = new TimestampFormatter();
        }
        return formatter;
    }

    private IdAndVersionStreamVerifierListener createVerifierListener(Function<Long, Object> formatter) {
        if (options.ignoreTimestampsDuringRun) {
            return createCoincidentPrintStreamListener(formatter);
        } else {
            return createStandardPrintStreamListener(formatter);
        }
    }

    IdAndVersionStreamVerifierListener createStandardPrintStreamListener(Function<Long, Object> formatter) {
        return new PrintStreamOutputVersionStreamVerifierListener(System.err, formatter);
    }

    IdAndVersionStreamVerifierListener createCoincidentPrintStreamListener(Function<Long, Object> formatter) {
        return new CoincidentFilteredStreamVerifierListener(new PrintStreamOutputVersionStreamVerifierListener(System.err, formatter));
    }


    public Scrutineer(ScrutineerCommandLineOptions options) {
        this.options = options;
    }

    private IdAndVersionFactory createIdAndVersionFactory() {
        return options.numeric ? LongIdAndVersion.FACTORY : StringIdAndVersion.FACTORY;
    }

    ElasticSearchIdAndVersionStream createElasticSearchIdAndVersionStream(ScrutineerCommandLineOptions options) {
        try {
            this.client = new ElasticSearchTransportClientFactory().getTransportClient(options);
            return new ElasticSearchIdAndVersionStream(new ElasticSearchDownloader(client, options.indexName, options.query, idAndVersionFactory), new ElasticSearchSorter(createSorter()), new IteratorFactory(idAndVersionFactory), SystemUtils.getJavaIoTmpDir().getAbsolutePath());
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    }


    private Sorter<IdAndVersion> createSorter() {
        SortConfig sortConfig = new SortConfig().withMaxMemoryUsage(DEFAULT_SORT_MEM);
        DataReaderFactory<IdAndVersion> dataReaderFactory = new IdAndVersionDataReaderFactory(idAndVersionFactory);
        DataWriterFactory<IdAndVersion> dataWriterFactory = new IdAndVersionDataWriterFactory();
        return new Sorter<IdAndVersion>(sortConfig, dataReaderFactory, dataWriterFactory, new NaturalComparator<IdAndVersion>());
    }

    JdbcIdAndVersionStream createJdbcIdAndVersionStream(ScrutineerCommandLineOptions options) {
        this.connection = initializeJdbcDriverAndConnection(options);
        return new JdbcIdAndVersionStream(connection, options.sql, idAndVersionFactory);
    }

    private Connection initializeJdbcDriverAndConnection(ScrutineerCommandLineOptions options) {
        try {
            Class.forName(options.jdbcDriverClass).newInstance();
            return DriverManager.getConnection(options.jdbcURL, options.jdbcUser, options.jdbcPassword);
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    }


    private static final int DEFAULT_SORT_MEM = 256 * 1024 * 1024;
    private final ScrutineerCommandLineOptions options;
    private IdAndVersionFactory idAndVersionFactory;
    private TransportClient client;
    private Connection connection;

}
